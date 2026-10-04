# Authentication

The AWS SDK for C++ supports multiple authentication strategies. This document covers the three you asked about: access key / secret key, IAM roles (instance profiles), and ARN-based resource auth.

## 1. Access Key / Secret Key Authentication

The simplest approach — pass credentials directly to the client constructor. Suitable for development and CI.

```cpp
#include <aws/s3/S3Client.h>
#include <aws/core/auth/AWSCredentials.h>

auto creds = Aws::Auth::AWSCredentials{
    "AKIAIOSFODNN7EXAMPLE",           // access key
    "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" // secret key
};
// optional session token for temporary credentials:
// creds.SetSessionToken("IQoJb3JpZ2luX2VjEA...");

Aws::Client::ClientConfiguration cfg;
cfg.region = "us-east-1";

auto client = std::make_shared<Aws::S3::S3Client>(creds, cfg);
```

**When to use**: Local dev, short-lived scripts, CI pipelines.

**Limits**:
- Credentials are **in-memory only** — lost on process restart.
- No rotation — you must recreate the client if credentials expire.
- `SimpleAWSCredentialsProvider` (used internally) never refreshes.

**Edge cases**:
- Empty or malformed keys → `AWSError` with `AccessDenied` or `SignatureDoesNotMatch`.
- Temporary credentials (from STS) must include session token via `SetSessionToken()`.

---

## 2. IAM Roles (Instance Profile / ECS / EKS)

Standard for production workloads on AWS infrastructure. The SDK's default credential provider chain discovers credentials automatically.

```cpp
#include <aws/s3/S3Client.h>

Aws::Client::ClientConfiguration cfg;
cfg.region = "us-east-1";
cfg.retryStrategy = std::make_shared<Aws::Client::DefaultRetryStrategy>();

// No credentials passed — uses default chain:
//   1. Environment vars (AWS_ACCESS_KEY_ID, etc.)
//   2. ~/.aws/credentials profile
//   3. STS web identity (AWS_ROLE_ARN + AWS_WEB_IDENTITY_TOKEN_FILE)
//   4. ECS task role (AWS_CONTAINER_CREDENTIALS_RELATIVE_URI)
//   5. EC2 instance profile (IMDS)
auto client = std::make_shared<Aws::S3::S3Client>(cfg);
```

### Explicit IAM Role Assumption (STS)

```cpp
#include <aws/sts/STSClient.h>
#include <aws/sts/model/AssumeRoleRequest.h>

Aws::Client::ClientConfiguration cfg;
cfg.region = "us-east-1";

auto sts = Aws::STS::STSClient{creds, cfg};
auto req = Aws::STS::Model::AssumeRoleRequest{}
    .WithRoleArn("arn:aws:iam::123456789012:role/MyAppRole")
    .WithRoleSessionName("app-session-1")
    .WithDurationSeconds(3600);

auto outcome = sts.AssumeRole(req);
if (!outcome.IsSuccess()) {
    // handle AWS::STS::STSErrors::AccessDenied, etc.
    throw std::runtime_error{outcome.GetError().GetMessage()};
}

auto roleCreds = outcome.GetResult().GetCredentials();
auto s3 = std::make_shared<Aws::S3::S3Client>(
    Aws::Auth::AWSCredentials{
        roleCreds.GetAccessKeyId(),
        roleCreds.GetSecretAccessKey(),
        roleCreds.GetSessionToken()
    },
    cfg
);
```

**When to use**: EC2, ECS, EKS, Lambda — any workload with an IAM role attached.

**Limits**:
- IMDS v1 falls back if v2 fails (disable by setting `AWS_EC2_METADATA_DISABLED=true` or using `DISABLE_INTERNAL_IMDSV1_CALLS` cmake flag).
- Default IMDS timeout: 1 second, 1 retry — may fail under heavy network load. Tune via `cfg.metadataServiceTimeout` and `cfg.metadataServiceNumAttempts`.
- STS assumed-role credentials expire (default 1 hour, max 12 hours). You must re-assume before expiry.
- The default credential provider chain **caches the winning provider**, not the credentials. If a provider returns empty credentials, the whole chain fails.
- `EnvironmentAWSCredentialsProvider` does **no caching** — reads env vars on every `GetAWSCredentials()` call.

**Edge cases**:

| Scenario | Symptom | Fix |
|----------|---------|-----|
| No IAM role attached | `AccessDenied` on all S3 calls | Attach role to EC2/ECS/EKS |
| IMDS blocked by firewall | Slow startup (timeout waiting for IMDS) | Use env vars or increase timeout |
| Clock skew on EC2 | `SignatureDoesNotMatch` | Enable NTP (`sudo chronyd` or `ntpd`) |
| STS AssumeRole policy denies | `AccessDenied` from STS | Check trust policy + permissions policy |
| Web identity token file missing | `STSCredentialsProvider` silently returns empty | Check `AWS_WEB_IDENTITY_TOKEN_FILE` path |
| ECS container without task role | `AccessDenied` | Add `TaskRoleArn` to ECS task definition |
| Chain exhausted | Empty credentials, all calls fail | Verify at least one provider succeeds |

---

## 3. ARN-Based Resource Authentication

ARNs identify specific AWS resources (buckets, access points, etc.). S3 supports ARN-based addressing for buckets and access points. The SDK resolves the ARN at request time.

```cpp
#include <aws/s3/S3Client.h>
#include <aws/s3/model/PutObjectRequest.h>
#include <aws/s3/model/GetObjectRequest.h>
#include <aws/core/client/ClientConfiguration.h>
#include <aws/core/auth/AWSCredentials.h>

Aws::Client::ClientConfiguration cfg;
cfg.region = "us-east-1";  // must match the ARN's region
cfg.useArnRegion = true;    // allow ARN to specify a different region

auto client = std::make_shared<Aws::S3::S3Client>(creds, cfg);

// --- Bucket ARN ---
auto putReq = Aws::S3::Model::PutObjectRequest{}
    .WithBucket("arn:aws:s3:us-east-1:123456789012:accesspoint/my-access-point")
    .WithKey("data/file.bin")
    .WithBody(std::make_shared<Aws::StringStream>("hello from ARN"));

auto putOutcome = client->PutObject(putReq);

// --- Access Point ARN ---
auto getReq = Aws::S3::Model::GetObjectRequest{}
    .WithBucket("arn:aws:s3:us-east-1:123456789012:accesspoint/my-access-point")
    .WithKey("data/file.bin");

auto getOutcome = client->GetObject(getReq);
```

### Multi-Region Access Points (MRAP)

```cpp
cfg.region = "*";  // MRAP requires wildcard region
auto client = std::make_shared<Aws::S3::S3Client>(creds, cfg);

auto req = Aws::S3::Model::PutObjectRequest{}
    .WithBucket("arn:aws:s3::123456789012:accesspoint/mrap-alias")
    .WithKey("data/file.bin");
```

**When to use**: S3 Access Points, Multi-Region Access Points, cross-account bucket access.

**Limits**:
- Not all ARN types are supported by every API. See [S3 ARN support](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-arn-s3.html).
- Cross-partition ARNs are **not supported** (e.g., `aws-cn` partition ARN with `aws-standard` client) — SDK check at `S3ARNSource.vm:80`.
- `useArnRegion = false` (default) requires the client region to match the ARN's region.
- MRAP requires SigV4a signing, which depends on the CRT library. If CRT is not linked, MRAP fails with `SignatureDoesNotMatch`.
- FIPS + path-only mode is a **TODO** (`S3EndpointProviderTests.cpp:1150`) — currently may produce wrong endpoint.

**Edge cases**:

| Scenario | Symptom | Fix |
|----------|---------|-----|
| ARN region mismatch | `PermanentRedirect` | Set `cfg.useArnRegion = true` or use matching region |
| Access Point doesn't exist | `NoSuchBucket` | Verify ARN string, check access point exists |
| No permission on access point | `AccessDenied` | Check access point policy + IAM policy |
| MRAP without CRT | Signing errors | Build with CRT (`BUILD_DEPS=ON`) |
| FIPS + Access Point | Wrong endpoint | Known SDK limitation (TODO). Use standard endpoint |
| ARN with cross-partition | `ValidationError` | Not supported. Use same-partition resources |

---

## 4. Default Credential Provider Chain (Production Best Practice)

Combines all methods with automatic fallback:

```cpp
// No credentials argument = default chain
auto client = std::make_shared<Aws::S3::S3Client>(cfg);

// The chain (in order):
// 1. EnvironmentAWSCredentialsProvider
// 2. ProfileConfigFileAWSCredentialsProvider
// 3. STSAssumeRoleWebIdentityCredentialsProvider
// 4. GeneralHTTPCredentialsProvider (ECS)
// 5. InstanceProfileCredentialsProvider (EC2)
```

**Limits**:
- The chain caches the **first successful provider** and reuses it on subsequent calls. If that provider's credentials expire, the chain does NOT fall through to other providers — it returns empty credentials.
- Profile files are re-read every 5 minutes (`REFRESH_THRESHOLD`).
- IMDSv1 fallback is enabled by default. Disable with `DISABLE_INTERNAL_IMDSV1_CALLS=ON`.

---

## 5. Custom Credential Provider Chain

For full control over fallback behavior:

```cpp
#include <aws/core/auth/AWSCredentialsProviderChain.h>

auto chain = std::make_shared<Aws::Auth::AWSCredentialsProviderChain>();

// Add custom providers in priority order
chain->AddProvider(
    std::make_shared<Aws::Auth::EnvironmentAWSCredentialsProvider>());
chain->AddProvider(
    std::make_shared<Aws::Auth::ProfileConfigFileAWSCredentialsProvider>());
chain->AddProvider(
    std::make_shared<Aws::Auth::InstanceProfileCredentialsProvider>());

auto client = std::make_shared<Aws::S3::S3Client>(creds, cfg);
```

---

## Function Reference

| Class / Function | Header | Purpose |
|-----------------|--------|---------|
| `AWSCredentials(ak, sk, [token])` | `aws/core/auth/AWSCredentials.h` | Holds access/secret/session |
| `S3Client(creds, cfg)` | `aws/s3/S3Client.h` | Construct with explicit creds |
| `S3Client(cfg)` | `aws/s3/S3Client.h` | Default credential chain |
| `STSClient::AssumeRole()` | `aws/sts/STSClient.h` | STS role assumption |
| `AWSCredentialsProviderChain` | `aws/core/auth/AWSCredentialsProvider.h` | Pluggable provider chain |
| `EnvironmentAWSCredentialsProvider` | `aws/core/auth/AWSCredentialsProvider.h` | Reads env vars |
| `InstanceProfileCredentialsProvider` | `aws/core/auth/AWSCredentialsProvider.h` | EC2 IMDS |
| `ProfileConfigFileAWSCredentialsProvider` | `aws/core/auth/AWSCredentialsProvider.h` | `~/.aws/credentials` |
| `SSOCredentialsProvider` | `aws/core/auth/SSOCredentialsProvider.h` | AWS SSO |
| `GeneralHTTPCredentialsProvider` | `aws/core/auth/GeneralHTTPCredentialsProvider.h` | ECS / generic HTTP |
| `ClientConfiguration::useArnRegion` | `aws/core/client/ClientConfiguration.h` | Enable cross-region ARN |

---

## Exception & Error Summary

```cpp
// All S3 operations return an Outcome type:
using Outcome = Aws::S3::S3Client::PutObjectOutcome;
// .IsSuccess() == true  => access via .GetResult()
// .IsSuccess() == false => access via .GetError()

// Check error details:
auto err = outcome.GetError();
err.GetErrorType();          // S3Errors enum
err.GetMessage();            // Human-readable
err.GetExceptionName();      // e.g. "AccessDenied"
err.ShouldRetry();           // true for retryable errors
err.GetResponseCode();       // HTTP status code
```

### Common S3Errors Related to Auth

| S3Errors Enum | HTTP | Meaning | Action |
|---------------|------|---------|--------|
| `ACCESS_DENIED` | 403 | No permission | Check IAM + resource policy |
| `INVALID_ACCESS_KEY_ID` | 403 | Access key unknown | Verify access key ID |
| `SIGNATURE_DOES_NOT_MATCH` | 403 | Bad signing | Check clock, secret key, region |
| `INVALID_SECURITY` | 403 | Bad sig or token expired | Refresh session token |
| `SLOW_DOWN` | 503 | Rate limited | Reduce throughput, backoff |
| `INTERNAL_ERROR` | 500 | Transient S3 error | Retry with backoff |
| `NETWORK_CONNECTION` | N/A | Connection failed | Check network, proxy |
| `ENDPOINT_RESOLUTION_FAILURE` | N/A | Bad endpoint | Check region, ARN |
| `CLIENT_SIGNING_FAILURE` | N/A | Signing internal error | Check CRT, clock |
