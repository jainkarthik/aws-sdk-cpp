# S3 PUT & GET with Verification

This document covers single-object upload and download using `PutObject` and `GetObject`, plus checksum and ETag-based verification.

## 1. Simple PUT (Upload)

```cpp
#include <aws/s3/S3Client.h>
#include <aws/s3/model/PutObjectRequest.h>
#include <aws/core/utils/memory/stl/AWSStringStream.h>
#include <fstream>

auto client = std::make_shared<Aws::S3::S3Client>(cfg);

// --- Upload from string ---
auto putReq = Aws::S3::Model::PutObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("hello.txt")
    .WithBody(std::make_shared<Aws::StringStream>("Hello, S3!"));

auto outcome = client->PutObject(putReq);
if (!outcome.IsSuccess()) {
    // Handle: NoSuchBucket, AccessDenied, EntityTooLarge, etc.
    throw std::runtime_error{
        "PutObject failed: " + outcome.GetError().GetMessage()
    };
}
auto etag = outcome.GetResult().GetETag(); // server-side ETag
```

### Upload from file

```cpp
#include <aws/core/utils/memory/stl/AWSStringStream.h>
#include <aws/core/utils/memory/AWSMemory.h>  // for Aws::New

auto putReq = Aws::S3::Model::PutObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("data/file.bin");

auto bodyStream = Aws::MakeShared<Aws::FStream>(
    "PutObjectStream",
    "/path/to/local/file.bin",
    std::ios_base::in | std::ios_base::binary);
putReq.SetBody(bodyStream);
putReq.SetContentLength(
    static_cast<long long>(bodyStream->tellg()));

auto outcome = client->PutObject(putReq);
```

### Upload with checksum (CRC64-NVME, new default)

```cpp
auto putReq = Aws::S3::Model::PutObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("data/file.bin")
    .WithChecksumAlgorithm(Aws::S3::Model::ChecksumAlgorithm::CRC64NVME);
// SDK computes and sends CRC64-NVME automatically
auto outcome = client->PutObject(putReq);
```

**Limits**:
- Max single-PUT object size: **5 GiB**. Larger files require multipart upload.
- Default checksum algorithm changed from MD5 to CRC64-NVME. Some third-party S3-compatible stores don't support CRC64-NVME — use `ChecksumAlgorithm::NONE` to fall back to MD5.
- `PutObject` overwrites an existing object with the same key — no versioning warning.
- Request body must be a seekable `std::istream` (wrapping `std::ifstream` or `std::stringstream`).

**Edge cases**:

| Situation | Behavior | Mitigation |
|-----------|----------|------------|
| File doesn't exist locally | `istream` failbit set → empty PUT | Check file exists, check `bodyStream->good()` |
| Object key with special chars | SDK URI-encodes automatically | No special handling needed |
| Bucket in different region | `PermanentRedirect` (301) | Set `cfg.region` to bucket's region |
| File modified during upload | Uploads the modified version | Snapshot file before upload |
| Very small files (< 1 byte) | Works, but wasteful | Consider aggregating (see [Aggregation](05-aggregation.md)) |

---

## 2. Simple GET (Download)

```cpp
#include <aws/s3/model/GetObjectRequest.h>
#include <aws/core/utils/stream/ResponseStream.h>
#include <fstream>

auto getReq = Aws::S3::Model::GetObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("hello.txt");

auto outcome = client->GetObject(getReq);
if (!outcome.IsSuccess()) {
    if (outcome.GetError().GetErrorType() ==
        Aws::S3::S3Errors::NO_SUCH_KEY) {
        // Object not found — handle gracefully
        return;
    }
    throw std::runtime_error{
        "GetObject failed: " + outcome.GetError().GetMessage()
    };
}

// Read into string
auto& body = outcome.GetResult().GetBody();
std::string content(
    std::istreambuf_iterator<char>(body),
    std::istreambuf_iterator<char>());

// Or write to file
auto& response = outcome.GetResult();
auto& stream = response.GetBody();
std::ofstream outFile("/path/to/save/file.bin",
                      std::ios::binary);
outFile << stream.rdbuf();
```

### Download to file with automatic retry

```cpp
#include <aws/core/utils/ratelimiter/DefaultRateLimiter.h>

auto limiter = std::make_shared<
    Aws::Utils::RateLimits::DefaultRateLimiter<>>(
        200 * 1024);  // 200 KB/s read limit

Aws::Client::ClientConfiguration cfg;
cfg.readRateLimiter = limiter;
cfg.retryStrategy = std::make_shared<
    Aws::Client::DefaultRetryStrategy>(
        3,    // max retries
        1000  // scale factor ms
    );

auto client = std::make_shared<Aws::S3::S3Client>(creds, cfg);
```

**Limits**:
- Entire object is downloaded into memory unless you stream to file.
- `GetObject` supports range downloads via `WithRange("bytes=0-1023")`.
- No automatic part-level retry — use `TransferManager` for large objects.
- Response body stream is **read-once** — non-seekable for large objects.

**Edge cases**:

| Situation | Behavior | Mitigation |
|-----------|----------|------------|
| Object doesn't exist | `NoSuchKey` (404) | Check with `HeadObject` first |
| Object deleted/versioned | Returns latest version | Use `WithVersionId()` |
| Object encrypted with KMS | Returns normally if client has KMS decrypt | Verify KMS key policy |
| Object > 2 GiB | Streams in chunks | Use range GET or multipart download |
| Network drops mid-stream | Partial download | Use `TransferManager` with retry |
| Object modified during GET | Reads original version (eventually consistent) | Use `IfMatch` / `IfNoneMatch` preconditions |

---

## 3. Verification

### ETag Comparison (Quick Check)

```cpp
#include <aws/s3/model/HeadObjectRequest.h>

// Step 1: Upload and capture ETag
auto putOutcome = client->PutObject(putReq);
auto serverEtag = putOutcome.GetResult().GetETag();

// Step 2: HeadObject to verify
auto headReq = Aws::S3::Model::HeadObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("hello.txt")
    .WithIfMatch(serverEtag); // conditional: fail if ETag changed

auto headOutcome = client->HeadObject(headReq);
if (!headOutcome.IsSuccess()) {
    if (headOutcome.GetError().GetErrorType() ==
        Aws::S3::S3Errors::PRECONDITION_FAILED) {
        // ETag changed — object was modified since upload
    }
}
```

### Client-Side Checksum Verification (CRC64-NVME)

```cpp
#include <aws/core/utils/crypto/CRC64NVME.h>
#include <aws/s3/model/PutObjectRequest.h>
#include <aws/s3/model/GetObjectRequest.h>

// --- Upload with CRC64-NVME ---
auto putReq = Aws::S3::Model::PutObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("data.bin")
    .WithChecksumAlgorithm(
        Aws::S3::Model::ChecksumAlgorithm::CRC64NVME);
// SDK computes CRC64-NVME automatically and includes it
auto putOutcome = client->PutObject(putReq);
if (!putOutcome.IsSuccess()) { /* handle */ }
auto receivedCrc64 = putOutcome.GetResult()
    .GetChecksumCRC64NVME();

// --- Download and verify ---
auto getOutcome = client->GetObject(getReq);
if (!getOutcome.IsSuccess()) { /* handle */ }

auto& body = getOutcome.GetResult().GetBody();
auto serverCrc64 = getOutcome.GetResult()
    .GetChecksumCRC64NVME();

if (!serverCrc64.empty() &&
    serverCrc64 != receivedCrc64) {
    // Data integrity check failed!
}

// Compute local CRC64-NVME for independent verification
auto localCrc = Aws::Utils::Crypto::CRC64NVME{};
// feed stream bytes into localCrc...
auto computed = localCrc.GetHash().GetHashString();
if (computed != serverCrc64) {
    // Object was corrupted in flight or at rest
}
```

### Full File Hash Verification (SHA-256)

```cpp
#include <aws/core/utils/crypto/Sha256.h>
#include <aws/core/utils/crypto/Sha256HMAC.h>
#include <aws/core/utils/HashingUtils.h>

// Compute local SHA-256
auto sha256 = Aws::Utils::Crypto::Sha256{};
std::ifstream file("/path/to/file.bin",
                   std::ios::binary);
auto localHash = sha256.Calculate(file);
auto localHashStr = Aws::Utils::HashingUtils::HexEncode(
    localHash.GetResult());

// HeadObject gets the server's ETag (MD5 for non-multipart)
auto headReq = Aws::S3::Model::HeadObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("file.bin");

auto headOutcome = client->HeadObject(headReq);
auto etag = headOutcome.GetResult().GetETag();
// ETag is quoted, e.g., "\"abc123...\"" — strip quotes
if (etag.size() >= 2 && etag.front() == '"' && etag.back() == '"') {
    etag = etag.substr(1, etag.size() - 2);
}
```

**Limits of ETag verification**:
- ETag for multipart uploads is **NOT** an MD5 of the whole object — it's a composite hash. Cannot be used for integrity verification.
- ETag for SSE-KMS/SSE-C encrypted objects is **NOT** the MD5 — it's a multipart ETag-like string.
- For multipart or encrypted objects, use CRC64-NVME or client-side SHA-256 instead.

### Verification with GetObjectAttributes

```cpp
#include <aws/s3/model/GetObjectAttributesRequest.h>

auto attrReq = Aws::S3::Model::GetObjectAttributesRequest{}
    .WithBucket("my-bucket")
    .WithKey("file.bin")
    .WithObjectAttributes({
        Aws::S3::Model::ObjectAttributes::ETag,
        Aws::S3::Model::ObjectAttributes::Checksum,
        Aws::S3::Model::ObjectAttributes::ObjectSize
    });

auto attrOutcome = client->GetObjectAttributes(attrReq);
if (attrOutcome.IsSuccess()) {
    auto result = attrOutcome.GetResult();
    auto size = result.GetObjectSize();
    auto etag = result.GetETag();
    auto crc64 = result.GetChecksum().GetChecksumCRC64NVME();
    // Compare against local values
}
```

---

## 4. Function Reference

| Function | Class | Return Type | Description |
|----------|-------|-------------|-------------|
| `PutObject(req)` | `S3Client` / `S3CrtClient` | `PutObjectOutcome` | Upload object (max 5 GiB) |
| `GetObject(req)` | `S3Client` / `S3CrtClient` | `GetObjectOutcome` | Download entire object |
| `HeadObject(req)` | `S3Client` / `S3CrtClient` | `HeadObjectOutcome` | Get metadata without body |
| `GetObjectAttributes(req)` | `S3Client` / `S3CrtClient` | `GetObjectAttributesOutcome` | Get checksum + size + ETag |
| `ObjectKey::SetChecksumAlgorithm()` | `PutObjectRequest` | `void` | Set CRC32/CRC32C/SHA1/SHA256/CRC64NVME |

## 5. Exception & Edge Case Matrix

```cpp
auto outcome = client->PutObject(req);
if (!outcome.IsSuccess()) {
    auto& err = outcome.GetError();
    switch (err.GetErrorType()) {
    case Aws::S3::S3Errors::NO_SUCH_BUCKET:
        // Create bucket or fix name
        break;
    case Aws::S3::S3Errors::ACCESS_DENIED:
        // Check IAM policy
        break;
    case Aws::S3::S3Errors::ENTITY_TOO_LARGE:
        // Object > 5 GiB — use multipart upload
        break;
    case Aws::S3::S3Errors::SLOW_DOWN:
        // Reduce concurrency / rate
        std::this_thread::sleep_for(
            std::chrono::seconds(1));
        // retry
        break;
    case Aws::S3::S3Errors::INTERNAL_FAILURE:
        // Transient — retry with backoff
        break;
    case Aws::S3::S3Errors::NETWORK_CONNECTION:
        // Check connectivity, then retry
        break;
    default:
        // Unexpected — log full error
        break;
    }
}
```
