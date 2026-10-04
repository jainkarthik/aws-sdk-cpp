# Aggregation Patterns for Small Files

When handling many small files (< 5 MiB), per-object overhead (both in cost and latency) becomes significant. This document covers **aggregation** (packing small files before upload) and **extraction** (unpacking on download).

## 1. Upload: Aggregate Small Files into a Single S3 Object

### 1a. Tar-Based Aggregation (POSIX style)

Pack multiple small files into a single `.tar` archive, then upload as one object.

```cpp
#include <aws/s3/model/PutObjectRequest.h>
#include <array>
#include <span>

// --- Simulate tar packing (conceptual — use libtar for production) ---
// Files are aggregated into a single in-memory buffer and uploaded
// as one S3 PUT. This reduces 100 PUT requests → 1 PUT.

auto packFiles = [](std::span<const std::string> files)
    -> std::shared_ptr<Aws::IOStream> {
    auto buffer = std::make_shared<Aws::StringStream>();
    // Production: use libtar or libarchive with zstd compression
    //   tar -cf - file1 file2 file3 | zstd -c
    // For each file, write:
    //   [header: 512 bytes of metadata]
    //   [content: padded to 512 bytes]
    // Reference: POSIX tar format (ustar)
    for (const auto& f : files) {
        std::ifstream in(f, std::ios::binary);
        // ... pack header + data ...
        buffer->write(/* header */);
        buffer->write(/* content */);
    }
    return buffer;
};

auto body = packFiles({"file1.txt", "file2.txt", "file3.txt"});

auto putReq = Aws::S3::Model::PutObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("archive/batch-20240625.tar.zst")
    .WithBody(body);

auto outcome = client->PutObject(putReq);
```

**Limits**:
- One S3 object max 5 TiB — unlikely to hit with small files.
- Tar does not natively support compression. Pair with zstd for space savings.
- Large archives delay first-byte latency for the consumer (must download entire archive).

### 1b. Zip-Based Aggregation (libzip / miniz)

```cpp
// Using a library like libzip:
//   zip_source_t* src = zip_source_buffer_create(data, len, 0, err);
//   zip_file_add(zip, "file1.txt", src);
// Then upload the resulting zip buffer to S3.
```

**When to use each format**:

| Format | Pros | Cons |
|--------|------|------|
| Tar + zstd | Fast compression, streaming-friendly | No random access (must decompress entire archive) |
| Zip | Random access, built-in CRC, widely supported | Slower compression, larger headers |
| Custom (raw concatenation) | Lowest overhead | No structure, consumer must know layout |

### 1c. Metadata Indexing

Store a manifest alongside the archive so consumers can find files:

```cpp
// Upload manifest after archive
auto manifestReq = Aws::S3::Model::PutObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("archive/batch-20240625.manifest.json")
    .WithBody(std::make_shared<Aws::StringStream>(
        R"({
            "archive": "archive/batch-20240625.tar.zst",
            "format": "tar+zstd",
            "files": [
                {"name": "file1.txt", "offset": 0, "size": 1234},
                {"name": "file2.txt", "offset": 1024, "size": 5678}
            ],
            "created": "2024-06-25T12:00:00Z"
        })"
    ));

client->PutObject(manifestReq);
```

**Limits**:
- The manifest is a separate S3 object — extra GET to read it before download.
- Offsets in the archive depend on the packing tool's output format.

---

## 2. Download: Extract Small Files from S3

### 2a. Download and Unpack Archive

```cpp
#include <aws/s3/model/GetObjectRequest.h>

// Step 1: Download the aggregate archive
auto getReq = Aws::S3::Model::GetObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("archive/batch-20240625.tar.zst");

auto outcome = client->GetObject(getReq);
if (!outcome.IsSuccess()) {
    throw std::runtime_error{
        "Download archive: " + outcome.GetError().GetMessage()
    };
}

// Step 2: Read full archive into memory
auto& body = outcome.GetResult().GetBody();
std::string archiveData(
    std::istreambuf_iterator<char>(body),
    std::istreambuf_iterator<char>());

// Step 3: Extract (using libarchive or similar)
// tar --zstd -xf archive.tar.zst
// Or programmatically with libarchive:
//   struct archive* a = archive_read_new();
//   archive_read_support_filter_zstd(a);
//   archive_read_support_format_tar(a);
//   archive_read_open_memory(a, archiveData.data(), archiveData.size());
//   while (archive_read_next_header(a, &entry) == ARCHIVE_OK) {
//       const void* buff; size_t size; int64_t offset;
//       archive_read_data_block(a, &buff, &size, &offset);
//       // write extracted file to disk
//   }
//   archive_read_close(a);
//   archive_read_free(a);
```

### 2b. Parallel Download + Local Aggregation

Instead of downloading one archive, download multiple small objects in parallel and aggregate locally:

```cpp
#include <aws/s3/model/GetObjectRequest.h>

std::vector<std::string> keys = {
    "logs/file1.log", "logs/file2.log", "logs/file3.log"
};
std::mutex aggregateMutex;
std::ofstream aggregateFile("/path/to/aggregated.log",
                            std::ios::binary);

auto downloadOne = [&](const std::string& key) -> bool {
    auto req = Aws::S3::Model::GetObjectRequest{}
        .WithBucket("my-bucket")
        .WithKey(key);

    auto outcome = client->GetObject(req);
    if (!outcome.IsSuccess()) return false;

    auto& body = outcome.GetResult().GetBody();
    std::string content(
        std::istreambuf_iterator<char>(body),
        std::istreambuf_iterator<char>());

    std::lock_guard lock(aggregateMutex);
    aggregateFile.write(content.data(), content.size());
    aggregateFile.put('\n');  // delimiter between files
    return true;
};

// Launch parallel downloads
std::vector<std::future<bool>> futures;
for (const auto& key : keys) {
    futures.push_back(std::async(std::launch::async,
        downloadOne, key));
}

bool allOk = true;
for (auto& f : futures) allOk &= f.get();
```

### 2c. Range-Based Partial Extraction from Archive

If you only need one file from a large archive, you can avoid downloading the entire archive by using a manifest with offsets and S3 range GETs:

```cpp
// Step 1: Download manifest (small JSON)
auto manifestReq = Aws::S3::Model::GetObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("archive/batch-20240625.manifest.json");
auto manifestOutcome = client->GetObject(manifestReq);

// Step 2: Parse manifest to get file offset + size
// (from metadata JSON)

// Step 3: Range GET for specific file within archive
auto rangeReq = Aws::S3::Model::GetObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("archive/batch-20240625.tar.zst")
    .WithRange("bytes=0-1233");  // offset 0, size 1234

auto rangeOutcome = client->GetObject(rangeReq);
// This only downloads the bytes for file1.txt
// Note: tar archives have 512-byte headers between files,
// so you need to parse the tar format to skip headers.
```

---

## 3. Per-Object vs Aggregated: When to Use

| Condition | Recommended Strategy |
|-----------|---------------------|
| File count < 100, each > 1 MiB | Upload individually (parallel) |
| File count > 1,000, each < 100 KiB | Aggregate into tar/zip archives |
| Files are logs — append-only | Aggregate on a timer (every N files or N seconds) |
| Need random access to individual files | Upload individually (or zip for random access) |
| Network latency dominates (> 50 ms per request) | Aggregate aggressively (reduces round-trips) |
| S3 PUT request cost is a concern | Aggregate (each PUT costs, regardless of size) |

### Cost Comparison (us-east-1, 2024)

| Strategy | # S3 PUTs | Cost (PUTs) | Storage (100 KiB × 10k files) | Total |
|----------|-----------|-------------|-------------------------------|-------|
| Individual upload | 10,000 | $0.05 | ~1 GiB = $0.023/mo | ~$0.073 + 10,000 round trips |
| Tar + zstd (100-file batches) | 100 | $0.0005 | ~200 MiB = $0.005/mo | ~$0.0055 + 100 round trips |

---

## 4. Limits & Edge Cases for Aggregation

| Situation | Behavior | Mitigation |
|-----------|----------|------------|
| Archive > 5 GiB | Single PUT fails (`EntityTooLarge`) | Use multipart upload for the archive itself |
| Lossy compression | Data corruption risk | Use zstd with integrity check (`--check`) |
| File modified during aggregation | Stale data in archive | Snapshot files before packing |
| Incomplete archive download | Partial data, no recovery | Use CRC64-NVME checksum on archive object |
| Tar with very long filenames (> 255 bytes) | Truncation in POSIX tar | Use GNU tar extension (`@LongLink`) or shorten names |
| Unicode filenames in tar | Encoding issues | Use UTF-8 consistently, verify on extraction |
| Very large archive (> 4 GiB) | Memory pressure holding entire archive | Stream extract (pipe `GetObject` → `archive_read`) |
| S3 eventual consistency after upload | GET may miss recently-uploaded archive | Write manifest after archive is confirmed via `WaitUntilCompleted` |

---

## 5. Function Reference

| Function | Class | Purpose |
|----------|-------|---------|
| `PutObject(req)` | `S3Client` / `S3CrtClient` | Upload aggregate as single object |
| `GetObject(req)` | `S3Client` / `S3CrtClient` | Download entire aggregate |
| `GetObject(req).WithRange()` | `GetObjectRequest` | Partial download from aggregate |
| `HeadObject(req)` | `S3Client` / `S3CrtClient` | Get aggregate metadata (size, checksum) |
| `TransferManager::UploadFile()` | `TransferManager` | Auto-multipart for large archives |
| `S3CrtClient::PutObject()` | `S3CrtClient` | CRT-accelerated aggregate upload |
