# AWS SDK for C++ Developer Guide — S3 Transfers

## Table of Contents

| # | Document | Covers |
|---|----------|--------|
| 01 | [Authentication](01-authentication.md) | ARN-based, access key, IAM role, credential provider chain, STS federation |
| 02 | [S3 Put & Get](02-s3-put-get.md) | Single-object PUT/GET, checksum verification, ETag validation, error handling |
| 03 | [Multipart Transfers](03-multipart.md) | Multipart upload/download, chunking strategy, part lifecycle, retry |
| 04 | [HPC & Parallel Transfers](04-hpc-transfers.md) | CRT client, concurrent transfers, throughput tuning, memory budgeting |
| 05 | [Aggregation Patterns](05-aggregation.md) | Batch small-files upload, zip/tar extraction, parallel download aggregation |

## SDK Version

This guide targets **AWS SDK for C++ 1.11.835** (C++17/20 compatible).

## Quick Reference: Key Functions

### Single-Object Operations

| Operation | Function | Class | Max Object Size |
|-----------|----------|-------|----------------|
| Upload | `PutObject()` | `S3Client`, `S3CrtClient` | 5 GiB |
| Download | `GetObject()` | `S3Client`, `S3CrtClient` | Unlimited (streaming) |
| Metadata | `HeadObject()` | `S3Client`, `S3CrtClient` | N/A |
| Verify | `HeadObject()` → compare ETag/checksum | `S3Client` | N/A |

### Multipart Operations

| Operation | Function | Class | Part Size Range |
|-----------|----------|-------|----------------|
| Initiate | `CreateMultipartUpload()` | `S3Client`, `S3CrtClient` | N/A |
| Upload Part | `UploadPart()` | `S3Client`, `S3CrtClient` | 5 MiB – 5 GiB |
| Upload Copy | `UploadPartCopy()` | `S3Client`, `S3CrtClient` | same as UploadPart |
| Complete | `CompleteMultipartUpload()` | `S3Client`, `S3CrtClient` | N/A |
| Abort | `AbortMultipartUpload()` | `S3Client`, `S3CrtClient` | N/A |
| List Parts | `ListParts()` | `S3Client`, `S3CrtClient` | N/A |

### High-Level Transfers

| Operation | Function | Class | Notes |
|-----------|----------|-------|-------|
| Upload file | `UploadFile()` | `TransferManager` | Auto multipart > bufferSize |
| Download file | `DownloadFile()` | `TransferManager` | Sequential by default |
| Retry upload | `RetryUpload()` | `TransferManager` | Re-sends only failed parts |
| Retry download | `RetryDownload()` | `TransferManager` | Re-fetches only failed parts |
| Directory upload | `UploadDirectory()` | `TransferManager` | Async, needs `transferInitiatedCallback` |
| Directory download | `DownloadToDirectory()` | `TransferManager` | Async, needs `transferInitiatedCallback` |

### CRT High-Performance Operations

| Operation | Function | Class | Notes |
|-----------|----------|-------|-------|
| Upload | `PutObject()` | `S3CrtClient` | Uses CRT pool, auto-multipart |
| Download | `GetObject()` | `S3CrtClient` | Flow-controlled, parallel |
| Config | `S3CrtClientConfiguration` | — | `partSize`, `throughputTargetGbps`, `downloadMemoryUsageWindow` |

## Hard Limits Summary

| Limit | Value | Enforced By |
|-------|-------|-------------|
| Max single-put object size | 5 GiB | S3 API |
| Min multipart part size | 5 MiB | S3 API |
| Max multipart part size | 5 GiB | S3 API |
| Max parts per upload | 10,000 | S3 API |
| Max object size (multipart) | 5 TiB | S3 API (10k × 5 GiB) |
| Max multipart uploads in-flight | 100 per bucket | S3 API (soft) |
| Min part size for TransferManager | 5 MiB | `TransferManagerConfiguration::bufferSize` default |
| Max TransferManager buffer heap | 50 MiB default | `transferBufferMaxHeapSize` |
| S3CrtClient default part size | 8 MiB | `S3CrtClientConfiguration::partSize` |
| S3CrtClient throughput target | 10 Gbps default | `throughputTargetGbps` |
| Presigned URL max lifetime | 7 days (604,800 s) | S3 API (via `AWSClient`) |
| Max S3CrtClient connections | 25 default | `ClientConfiguration::maxConnections` |
| Max retries (default) | 10 | `DefaultRetryStrategy` |

## Error Category Reference

| Error | Cause | Typical Handling |
|-------|-------|-----------------|
| `NoSuchBucket` | Bucket doesn't exist | Check bucket name, create bucket |
| `NoSuchKey` | Object key not found | Check key path, list objects |
| `AccessDenied` | No permission / invalid creds | Verify auth, IAM policy |
| `SignatureDoesNotMatch` | Bad signing / clock skew | Check system time, credentials |
| `SlowDown` | Rate limited | Exponential backoff, reduce concurrency |
| `EntityTooLarge` | Object > 5 GiB in single PUT | Switch to multipart upload |
| `InvalidPart` | Upload part ETag mismatch | Retry the part |
| `InvalidPartOrder` | Parts completed out of order | Complete in ascending part number |
| `RequestTimeout` | Part upload timed out | Increase `requestTimeoutMs`, retry |
| `InternalError` | S3 transient error | Retry with backoff |
| `BucketAlreadyExists` | Global bucket name taken | Use unique bucket name |
| `KMSFailure` | KMS key access denied | Check KMS key policy |
| `PermanentRedirect` | Wrong region | Update `ClientConfiguration::region` |

## Common Edge Cases

1. **Clock skew**: SigV4 signing requires system time within 5 minutes of AWS. Use NTP.
2. **Non-seekable streams**: Multipart upload requires seekable streams. Wrap in `Aws::IOStream` with `Aws::StringStream` backing.
3. **Network interruptions**: Use `RetryUpload`/`RetryDownload` with exponential backoff.
4. **Orphaned multipart uploads**: Always call `AbortMultipartUpload` on failure, or set lifecycle policy on S3 bucket.
5. **Very large files (>50 GiB)**: Increase `partSize` to reduce part count (max 10k parts).
6. **Small files (<5 MiB)**: Use single `PutObject` — multipart overhead is wasteful.
7. **Checksum fallback**: New default is CRC64-NVME. Third-party S3-compatible stores may not support it; fall back to MD5 via `ChecksumAlgorithm::NONE` (SDK falls back to MD5).
8. **Windows MAX_PATH**: Object paths + local paths can exceed 260 chars. Use `\\?\` prefix or shorten local path.
9. **Memory exhaustion**: With many concurrent transfers, `transferBufferMaxHeapSize` acts as a cap. For CRT, `downloadMemoryUsageWindow` controls flow control.
10. **AWS-LC Apple Store**: AWS-LC AES-GCM uses private APIs rejected by Apple App Store. Set `AWS_APPSTORE_SAFE=ON` (disables AES-GCM path).
