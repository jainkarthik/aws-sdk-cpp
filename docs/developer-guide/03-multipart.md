# Multipart Upload & Download with Chunking

Multipart transfers are required for objects > 5 GiB and recommended for objects > 100 MiB for improved throughput and resilience.

## 1. Manual Multipart Upload

The SDK provides low-level `CreateMultipartUpload`, `UploadPart`, and `CompleteMultipartUpload` calls.

```cpp
#include <aws/s3/model/CreateMultipartUploadRequest.h>
#include <aws/s3/model/UploadPartRequest.h>
#include <aws/s3/model/CompleteMultipartUploadRequest.h>
#include <aws/s3/model/CompletedPart.h>
#include <aws/core/utils/memory/stl/AWSVector.h>

auto client = std::make_shared<Aws::S3::S3Client>(cfg);
constexpr uint64_t PART_SIZE = 8 * 1024 * 1024;  // 8 MiB

// --- Step 1: Initiate ---
auto createReq = Aws::S3::Model::CreateMultipartUploadRequest{}
    .WithBucket("my-bucket")
    .WithKey("large-file.bin")
    .WithChecksumAlgorithm(
        Aws::S3::Model::ChecksumAlgorithm::CRC64NVME);

auto createOutcome = client->CreateMultipartUpload(createReq);
if (!createOutcome.IsSuccess()) {
    throw std::runtime_error{
        "CreateMultipartUpload: " +
        createOutcome.GetError().GetMessage()
    };
}
auto uploadId = createOutcome.GetResult().GetUploadId();

// --- Step 2: Upload parts ---
std::ifstream file("/path/to/large-file.bin", std::ios::binary);
Aws::Vector<Aws::S3::Model::CompletedPart> completedParts;
int partNumber = 1;

while (file) {
    auto buffer = std::make_shared<Aws::StringStream>();
    std::vector<char> chunk(PART_SIZE);
    file.read(chunk.data(), PART_SIZE);
    auto bytesRead = file.gcount();
    if (bytesRead == 0) break;

    buffer->write(chunk.data(), bytesRead);

    auto uploadReq = Aws::S3::Model::UploadPartRequest{}
        .WithBucket("my-bucket")
        .WithKey("large-file.bin")
        .WithPartNumber(partNumber)
        .WithUploadId(uploadId)
        .WithBody(buffer);

    auto uploadOutcome = client->UploadPart(uploadReq);
    if (!uploadOutcome.IsSuccess()) {
        // Abort on failure
        client->AbortMultipartUpload(
            Aws::S3::Model::AbortMultipartUploadRequest{}
                .WithBucket("my-bucket")
                .WithKey("large-file.bin")
                .WithUploadId(uploadId));
        throw std::runtime_error{
            "UploadPart " + std::to_string(partNumber) +
            ": " + uploadOutcome.GetError().GetMessage()
        };
    }

    completedParts.push_back(
        Aws::S3::Model::CompletedPart{}
            .WithPartNumber(partNumber)
            .WithETag(uploadOutcome.GetResult().GetETag()));
    partNumber++;
}

// --- Step 3: Complete ---
auto completeReq = Aws::S3::Model::CompleteMultipartUploadRequest{}
    .WithBucket("my-bucket")
    .WithKey("large-file.bin")
    .WithUploadId(uploadId)
    .WithMultipartUpload(
        Aws::S3::Model::CompletedMultipartUpload{}
            .WithParts(completedParts));

auto completeOutcome = client->CompleteMultipartUpload(completeReq);
if (!completeOutcome.IsSuccess()) {
    // Handle failure — parts may still exist in S3
    client->AbortMultipartUpload(
        Aws::S3::Model::AbortMultipartUploadRequest{}
            .WithBucket("my-bucket")
            .WithKey("large-file.bin")
            .WithUploadId(uploadId));
    throw std::runtime_error{
        "CompleteMultipartUpload: " +
        completeOutcome.GetError().GetMessage()
    };
}
```

**Limits**:
- Part size range: **5 MiB – 5 GiB** (last part can be smaller).
- Max parts per upload: **10,000**.
- With max part size (5 GiB) × 10,000 parts = **max object 5 TiB**.
- Parts must be uploaded in order or tracked manually for `CompleteMultipartUpload`.
- If you abort, already-uploaded parts remain in S3 and incur storage costs. Always call `AbortMultipartUpload` after aborting.

**Edge cases**:

| Situation | Behavior | Mitigation |
|-----------|----------|------------|
| Last part < 5 MiB | Allowed (only last part can be small) | Ensure prior parts are ≥ 5 MiB |
| Part ETag mismatch on complete | `InvalidPart` error | Retry the specific part |
| Parts completed out of order | `InvalidPartOrder` | Sort by part number before `Complete` |
| UploadId expires | Aborted — parts deleted by S3 lifecycle | Complete within 7 days (or configure lifecycle) |
| Part upload timeout | `RequestTimeout` | Increase `requestTimeoutMs`, retry part |
| Concurrent upload of same part | Last one wins | Use idempotency keys or sequential parts |
| File grows during upload | Parts 2-N offset drifts | Snapshot file size before starting |

---

## 2. TransferManager (Auto Multipart)

`TransferManager` handles part lifecycle, retry, and completion automatically.

```cpp
#include <aws/transfer/TransferManager.h>
#include <aws/transfer/TransferHandle.h>

// --- Setup ---
Aws::Transfer::TransferManagerConfiguration tmCfg{
    Aws::MakeShared<Aws::Utils::Threading::PooledThreadExecutor>(
        "executor", 8)  // 8 threads
};
tmCfg.s3Client = client;
tmCfg.bufferSize = 16 * 1024 * 1024;  // 16 MiB parts
tmCfg.transferBufferMaxHeapSize = 512 * 1024 * 1024;  // 512 MiB max
tmCfg.checksumAlgorithm =
    Aws::S3::Model::ChecksumAlgorithm::CRC64NVME;
tmCfg.validateChecksums = true;

// Callbacks
tmCfg.uploadProgressCallback =
    [](const auto*, const auto& handle) {
        std::print("Upload: {} / {}\n",
            handle->GetBytesTransferred(),
            handle->GetBytesTotalSize());
    };
tmCfg.errorCallback =
    [](const auto*, const auto& handle, const auto& error) {
        std::print("Error on {}: {}\n",
            handle->GetKey(), error.GetMessage());
    };

auto tm = Aws::Transfer::TransferManager::Create(tmCfg);

// --- Upload (auto multipart if > bufferSize) ---
auto uploadHandle = tm->UploadFile(
    "/path/to/large-file.bin",
    "my-bucket",
    "large-file.bin",
    "application/octet-stream",  // contentType
    Aws::Map<Aws::String, Aws::String>{}  // metadata
);
uploadHandle->WaitUntilCompleted();

if (uploadHandle->GetStatus() !=
    Aws::Transfer::TransferStatus::COMPLETED) {
    // Failed or cancelled — clean up
    if (uploadHandle->GetStatus() ==
        Aws::Transfer::TransferStatus::FAILED) {
        tm->AbortMultipartUpload(uploadHandle);
    }
}

// --- Download (always sequential) ---
auto downloadHandle = tm->DownloadFile(
    "my-bucket",
    "large-file.bin",
    "/path/to/save/large-file.bin"
);
downloadHandle->WaitUntilCompleted();

// --- Retry failed upload (only failed parts) ---
auto retryHandle = tm->RetryUpload(
    "/path/to/large-file.bin",
    uploadHandle);
retryHandle->WaitUntilCompleted();

// --- Graceful shutdown ---
tm->WaitUntilAllFinished(60'000);  // 60s timeout
```

### DownloadConfiguration

```cpp
Aws::Transfer::DownloadConfiguration dlCfg;
dlCfg.versionId = "abc123"; // specific version

auto handle = tm->DownloadFile(
    "my-bucket", "key", "/path/to/save",
    dlCfg);
```

**Limits**:
- `bufferSize` must be ≥ 5 MiB. Default: 5 MiB.
- `transferBufferMaxHeapSize` caps total allocated buffer memory across all transfers. Default: 50 MiB. Must be ≥ `bufferSize × poolSize` for `PooledThreadExecutor`.
- `WaitUntilCompleted()` blocks the calling thread.
- `TransferManager` returns `TransferHandle` immediately — operations run on executor threads.
- Download only supports single-threaded transfer internally. Use CRT for parallel downloads.

**Edge cases**:

| Situation | Behavior | Mitigation |
|-----------|----------|------------|
| Upload buffer too small | Memory allocation failures | Increase `transferBufferMaxHeapSize` |
| TransferManager outlives SDK scope | Crash on `ShutdownAPI` | Call `WaitUntilAllFinished()` before shutdown |
| File not found on disk | Upload reads 0 bytes into S3 | Verify file exists before `UploadFile` |
| Network drops mid-transfer | Parts fail → `FAILED` status | Call `RetryUpload` — only failed parts re-sent |
| Multipart upload abandoned | Parts accrue storage cost | Call `AbortMultipartUpload` or set S3 lifecycle |

---

## 3. Chunking Strategy

Chunking refers to how data is split across parts and how each part is signed (streaming chunked transfer encoding).

### Automatic Chunked Transfer Encoding

The SDK uses `aws-chunked` content encoding for PUT requests with trailing checksums. This is enabled automatically when:
- `ChecksumAlgorithm` is set to anything other than `NONE`
- Payload signing policy is `Always` or `RequestDependent`

```cpp
// Forces chunked encoding with trailing checksum
auto putReq = Aws::S3::Model::PutObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("file.bin")
    .WithChecksumAlgorithm(
        Aws::S3::Model::ChecksumAlgorithm::CRC64NVME);

// Payload signing policy in client config
// ::Never        — No chunking, no checksum trailer
// ::Always       — Chunked with signed trailer
// ::RequestDependent — auto based on request (default)
Aws::S3::S3ClientConfiguration s3Cfg;
s3Cfg.payloadSigningPolicy =
    Aws::Client::AWSAuthV4Signer::PayloadSigningPolicy::Always;
```

### Manual Chunk Size Control

For the CRT client, `partSize` controls the chunk size for multipart uploads:

```cpp
Aws::S3Crt::S3CrtClientConfiguration crtCfg;
crtCfg.partSize = 64 * 1024 * 1024; // 64 MiB chunks
// Larger chunks = fewer parts = less overhead
// But each failed part requires re-sending more data
```

### When to tune chunk size

| File Size | Recommended Part Size | # Parts | Rationale |
|-----------|----------------------|---------|-----------|
| < 50 MiB | N/A (single PUT) | 1 | Multipart overhead not worth it |
| 50 MiB – 1 GiB | 8–16 MiB | 6–128 | Good balance of parallelism vs overhead |
| 1–50 GiB | 16–64 MiB | 16–3,200 | Reduce part management overhead |
| 50–500 GiB | 64–256 MiB | 200–8,000 | Keep parts < 10k limit |
| 500 GiB – 5 TiB | 512 MiB – 5 GiB | 1,000–10,000 | Stay within 10k limit |

---

## 4. Function Reference

| Function | Class | Description |
|----------|-------|-------------|
| `CreateMultipartUpload(req)` | `S3Client` / `S3CrtClient` | Get uploadId for subsequent parts |
| `UploadPart(req)` | `S3Client` / `S3CrtClient` | Upload a single part (5 MiB – 5 GiB) |
| `UploadPartCopy(req)` | `S3Client` / `S3CrtClient` | Copy a part from another object |
| `CompleteMultipartUpload(req)` | `S3Client` / `S3CrtClient` | Assemble parts into final object |
| `AbortMultipartUpload(req)` | `S3Client` / `S3CrtClient` | Discard upload, remove stored parts |
| `ListParts(req)` | `S3Client` / `S3CrtClient` | List uploaded parts for an uploadId |
| `ListMultipartUploads(req)` | `S3Client` / `S3CrtClient` | List in-progress multipart uploads |
| `TransferManager::UploadFile()` | `TransferManager` | Auto multipart with retry |
| `TransferManager::DownloadFile()` | `TransferManager` | Single-stream download |
| `TransferManager::RetryUpload()` | `TransferManager` | Re-send only failed parts |
| `TransferManager::AbortMultipartUpload()` | `TransferManager` | Clean up failed upload |
| `WaitUntilCompleted()` | `TransferHandle` | Block until transfer finishes |
| `GetStatus()` | `TransferHandle` | COMPLETED / FAILED / CANCELED / ABORTED |
