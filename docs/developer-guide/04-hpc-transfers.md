# HPC & Parallel Transfers

For high-throughput and concurrent transfers, use the **S3 CRT client** (`S3CrtClient`), which provides:

- **Automatic multipart** with parallel part upload
- **Flow-controlled downloads** with configurable memory window
- **Throughput targeting** (set desired Gbps, CRT optimizes)
- **Connection pooling** with HTTP/2
- **Separate retry strategy** tuned for high concurrency

## 1. S3CrtClient — High-Performance Setup

```cpp
#include <aws/s3-crt/S3CrtClient.h>
#include <aws/s3-crt/S3CrtClientConfiguration.h>

Aws::S3Crt::S3CrtClientConfiguration crtCfg;
crtCfg.region = "us-east-1";
crtCfg.throughputTargetGbps = 25.0;  // target 25 Gbps
crtCfg.partSize = 64 * 1024 * 1024;  // 64 MiB per part
crtCfg.maxConnections = 100;         // connection pool depth
crtCfg.downloadMemoryUsageWindow = 256 * 1024 * 1024;  // 256 MiB flow control

// Retry config for CRT (separate from SDK retry)
crtCfg.crtRetryStrategyConfig.crtRetryStrategyType =
    Aws::S3Crt::S3CrtClientConfiguration::CrtRetryStrategyConfig::CrtRetryStrategyType::STANDARD;
crtCfg.crtRetryStrategyConfig.config.maxRetries = 5;
crtCfg.crtRetryStrategyConfig.config.scaleFactorMs = 1000;
crtCfg.crtRetryStrategyConfig.config.maxBackoffSecs = 30;

// Memory budget (0 = unlimited)
crtCfg.memoryLimitBytes = 2UL * 1024 * 1024 * 1024;  // 2 GiB

auto client = std::make_shared<Aws::S3Crt::S3CrtClient>(creds, crtCfg);
```

### Performance Tuning Parameters

| Parameter | Default | Recommended Range | Effect |
|-----------|---------|-------------------|--------|
| `partSize` | 8 MiB | 8 MiB – 256 MiB | Larger = fewer parts, less overhead; smaller = better parallel fill |
| `throughputTargetGbps` | 10.0 | 1.0 – 100.0 | CRT adjusts concurrency to hit target |
| `maxConnections` | 25 | 50 – 500 | Max concurrent HTTP connections |
| `downloadMemoryUsageWindow` | 0 | 256 MiB – 2 GiB | 0 = unlimited; >0 limits peak download memory |
| `memoryLimitBytes` | 0 | 1 GiB – 4 GiB | 0 = unlimited; caps total CRT memory |
| `multipartUploadThreshold` | `partSize` | `partSize` – 1 GiB | Files above this use multipart |

**Limits**:
- `throughputTargetGbps` is a **target**, not a guarantee. Actual throughput depends on network, instance type, and S3 throttling.
- `downloadMemoryUsageWindow` only applies to downloads. Upload memory is limited by `memoryLimitBytes`.
- `partSize` minimum is 5 MiB (CRT enforces). Values < 5 MiB are silently raised.
- `networkInterfaceNames` is **experimental and unstable** — avoid in production.

---

## 2. Parallel Uploads with S3CrtClient

The CRT client handles part-level parallelism automatically. Below is an example of object-level parallelism (uploading multiple objects concurrently).

```cpp
#include <aws/s3-crt/S3CrtClient.h>
#include <aws/s3/model/PutObjectRequest.h>
#include <aws/core/utils/threading/PooledThreadExecutor.h>
#include <list>
#include <future>
#include <syncstream>

auto client = std::make_shared<Aws::S3Crt::S3CrtClient>(creds, crtCfg);

// --- Parallel object upload (multiple files) ---
std::vector<std::string> files = {
    "file1.bin", "file2.bin", "file3.bin", "file4.bin"
};

auto executor = std::make_shared<
    Aws::Utils::Threading::PooledThreadExecutor>(files.size());

std::vector<std::future<bool>> futures;

for (const auto& f : files) {
    auto fut = std::async(std::launch::deferred, [&client, &f]() {
        auto body = std::make_shared<Aws::FStream>(
            f.c_str(), std::ios::in | std::ios::binary);
        if (!body->good()) return false;

        auto putReq = Aws::S3::Model::PutObjectRequest{}
            .WithBucket("my-bucket")
            .WithKey("uploads/" + f)
            .WithBody(body);

        auto outcome = client->PutObject(putReq);
        if (!outcome.IsSuccess()) {
            std::println(std::cerr, "Failed: {} — {}",
                f, outcome.GetError().GetMessage());
            return false;
        }
        std::println("Uploaded: {} (ETag: {})",
            f, outcome.GetResult().GetETag());
        return true;
    });
    futures.push_back(std::async(std::launch::async,
        [exec = executor, f = std::move(fut)]() mutable {
            return f.get();
        }));
}

int success = 0, failure = 0;
for (auto& f : futures) {
    (f.get() ? success : failure)++;
}
std::println("Result: {} succeeded, {} failed", success, failure);
```

**Rules for true parallelism**:
1. Each thread gets its own `S3CrtClient` or shares one (CRT is thread-safe for concurrent requests).
2. Use `std::async(std::launch::async)` or a thread pool — the SDK executor is for internal queuing.
3. Each concurrent request consumes a connection from the pool. Set `maxConnections` accordingly.
4. For maximum throughput, match thread count to `throughputTargetGbps / single-stream-throughput`.

---

## 3. Parallel Downloads with S3CrtClient

CRT downloads use the same connection pool and flow control.

```cpp
#include <aws/s3/model/GetObjectRequest.h>

auto getReq = Aws::S3::Model::GetObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("large-file.bin");

// CRT client performs parallel part-level download
// internally. Single GetObject call can saturate
// the connection pool.
auto outcome = client->GetObject(getReq);
if (outcome.IsSuccess()) {
    auto& body = outcome.GetResult().GetBody();
    std::ofstream out("/path/to/save/large-file.bin",
                      std::ios::binary);
    out << body.rdbuf();
}
```

### Range-Based Parallel Download (Manual Sharding)

For ultimate control, download byte ranges in parallel:

```cpp
#include <aws/s3/model/HeadObjectRequest.h>

auto headReq = Aws::S3::Model::HeadObjectRequest{}
    .WithBucket("my-bucket")
    .WithKey("large-file.bin");
auto headOutcome = client->HeadObject(headReq);
auto objSize = headOutcome.GetResult().GetContentLength();

constexpr uint64_t RANGE_SIZE = 128 * 1024 * 1024; // 128 MiB
uint64_t numRanges = (objSize + RANGE_SIZE - 1) / RANGE_SIZE;

std::vector<std::future<bool>> rangeFutures;
std::mutex writeMutex;
std::ofstream outFile("/path/to/save/large-file.bin",
                      std::ios::binary);

for (uint64_t i = 0; i < numRanges; ++i) {
    uint64_t start = i * RANGE_SIZE;
    uint64_t end = std::min(start + RANGE_SIZE - 1, objSize - 1);

    rangeFutures.push_back(std::async(std::launch::async,
        [client, i, start, end, &outFile, &writeMutex]() {
        auto rangeReq = Aws::S3::Model::GetObjectRequest{}
            .WithBucket("my-bucket")
            .WithKey("large-file.bin")
            .WithRange(
                "bytes=" + std::to_string(start) +
                "-" + std::to_string(end));

        auto outcome = client->GetObject(rangeReq);
        if (!outcome.IsSuccess()) return false;

        auto& body = outcome.GetResult().GetBody();
        std::string data(
            std::istreambuf_iterator<char>(body),
            std::istreambuf_iterator<char>());

        std::lock_guard lock(writeMutex);
        outFile.seekp(start);
        outFile.write(data.data(), data.size());
        return true;
    }));
}

// Wait for all ranges
bool allOk = true;
for (auto& f : rangeFutures) allOk &= f.get();
```

**Limits**:
- Range GET is most effective when combined with a large file and fast network (reduces per-range overhead).
- Each range GET is a separate HTTP request — there's overhead for very small ranges.
- Write ordering matters — use mutex + seekp or separate files + concatenate.

---

## 4. Concurrent Transfers with TransferManager + Custom Executor

`TransferManager` uses a thread executor for concurrent transfers when you use `PooledThreadExecutor`.

```cpp
#include <aws/transfer/TransferManager.h>
#include <aws/core/utils/threading/PooledThreadExecutor.h>

auto executor = Aws::MakeShared<
    Aws::Utils::Threading::PooledThreadExecutor>(
    "hpc-pool", 16);  // 16 threads

Aws::Transfer::TransferManagerConfiguration tmCfg{
    executor.get()  // raw pointer
};
tmCfg.s3Client = client;
tmCfg.transferBufferMaxHeapSize = 2048 * 1024 * 1024; // 2 GiB
tmCfg.bufferSize = 32 * 1024 * 1024; // 32 MiB parts

// Progress tracking
std::atomic<uint64_t> totalUploaded{0};
tmCfg.uploadProgressCallback =
    [&totalUploaded](const auto*, const auto& h) {
        totalUploaded += h->GetBytesTransferred();
    };

auto tm = Aws::Transfer::TransferManager::Create(tmCfg);

// Launch concurrent uploads
std::vector<std::shared_ptr<Aws::Transfer::TransferHandle>> handles;
for (const auto& [local, key] : fileMapping) {
    auto h = tm->UploadFile(
        local, "my-bucket", key, "application/octet-stream", {});
    handles.push_back(h);
}

// Wait for all
for (auto& h : handles) {
    h->WaitUntilCompleted();
    if (h->GetStatus() !=
        Aws::Transfer::TransferStatus::COMPLETED) {
        // Handle failure per-file
    }
}

tm->WaitUntilAllFinished();
```

---

## 5. Throughput Fine-Tuning Guide

### Network-Level Tuning

| Parameter | Where | Effect |
|-----------|-------|--------|
| `maxConnections` | `ClientConfiguration` | Pool depth for HTTP connections |
| `throughputTargetGbps` | `S3CrtClientConfiguration` | CRT self-tunes concurrency |
| `tcpKeepAliveIntervalMs` | `ClientConfiguration` | Keep idle connections alive |
| `lowSpeedLimit` | `ClientConfiguration` (Curl) | Abort slow transfers (bytes/sec) |

### TransferManager Tuning

| Parameter | Where | Effect |
|-----------|-------|--------|
| `bufferSize` | `TransferManagerConfiguration` | Part size for multipart |
| `transferBufferMaxHeapSize` | `TransferManagerConfiguration` | Total buffer memory cap |
| `PooledThreadExecutor(size)` | Constructor arg | Concurrent part upload threads |
| `uploadProgressCallback` | Configuration | Back-pressure monitoring |

### Memory Budget Estimates

| Scenario | Part Size | Concurrency | Memory Used |
|----------|-----------|-------------|-------------|
| Single file, low memory | 8 MiB | 1 | 8 MiB |
| Single large file | 64 MiB | 4 (CRT) | 256 MiB |
| Many small files (100×) | 5 MiB | 16 threads | 8 GiB total |
| HPC download, limited | 32 MiB | window=512 MiB | ≤ 512 MiB controlled |
| Max throughput | 64 MiB | 100 connections | unbounded (set `memoryLimitBytes`) |

### Instance Type Guidance

| AWS Instance | Network | Recommended `throughputTargetGbps` | `maxConnections` |
|-------------|---------|-----------------------------------|------------------|
| c5.large | Up to 10 Gbps | 10.0 | 50 |
| c5n.4xlarge | Up to 25 Gbps | 20.0 | 100 |
| c5n.18xlarge | Up to 100 Gbps | 75.0 | 500 |
| t3.medium | Up to 5 Gbps | 2.0 | 25 |
| Lambda (512 MB) | ~1 Gbps | 1.0 | 10 |

---

## 6. Function Reference

| Function / Class | Header | Purpose |
|-----------------|--------|---------|
| `S3CrtClient` | `aws/s3-crt/S3CrtClient.h` | CRT-accelerated S3 client |
| `S3CrtClientConfiguration` | `aws/s3-crt/S3CrtClientConfiguration.h` | Part size, throughput, memory tuning |
| `PooledThreadExecutor(n)` | `aws/core/utils/threading/PooledThreadExecutor.h` | Thread pool for parallel work |
| `TransferManager::UploadFile()` | `aws/transfer/TransferManager.h` | Concurrent upload via executor |
| `ClientConfiguration::maxConnections` | `aws/core/client/ClientConfiguration.h` | HTTP pool depth |
| `ClientConfiguration::readRateLimiter` | `aws/core/client/ClientConfiguration.h` | Bytes/sec read throttle |
| `ClientConfiguration::writeRateLimiter` | `aws/core/client/ClientConfiguration.h` | Bytes/sec write throttle |

---

## 7. Exception & Edge Case Matrix for HPC

| Scenario | Symptom | Root Cause | Fix |
|----------|---------|------------|-----|
| Throughput far below target | Slow transfers | `maxConnections` too low | Increase pool depth |
| Memory exhaustion | OOM kill | `memoryLimitBytes = 0` | Set memory budget |
| Connection pool exhausted | `NETWORK_CONNECTION` errors | Too many concurrent requests | Increase `maxConnections` or reduce concurrency |
| S3 throttling (503 SlowDown) | Transfer rate plummets | Exceeding bucket/prefix rate limits | Reduce concurrency, increase part size |
| CRT HTTP client not linked | Link errors at compile | `USE_CRT_HTTP_CLIENT=OFF` | Enable in CMake |
| Download memory spike | Crash on large file | `downloadMemoryUsageWindow = 0` | Set to a finite window |
| TransferManager outlives SDK | Use-after-free on shutdown | Missing `WaitUntilAllFinished()` | Call before `ShutdownAPI()` |
| Non-seekable stream with CRT | Upload fails | CRT needs seekable for content-length | Wrap in `Aws::StringStream` or use `CONTENT_LENGTH_CONFIGURATION::SKIP_CONTENT_LENGTH` |
