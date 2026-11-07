# aws

AWS for [Quarz](https://quarz.dev), written in Quarz over its standard
library: credentials (the SDKs' chain), request signing (Signature
Version 4, usable for any service), and service clients as modules of
this cog, starting with KMS.

```toml
# Cog.toml of an application
[dependencies]
AWS = { git = "https://github.com/quarz-lang/aws", tag = "v0.1.0" }
```

Work in progress: the design is issue #1. What exists:

### Credentials

```ruby
creds = AWS::Credentials.default   # a provider: Shareable, cached, refreshed
keys = creds.get                   # AWS::Credentials::Keys
keys.inspect                       # => "#<AWS::Credentials::Keys ASIA... expires 2026-11-07T12:00:00Z>"
```

- `AWS::Credentials::Keys(access_key_id, secret_access_key, session_token, expires_at)`:
  `inspect` and `to_s` show the access key id and the expiry only.
- Providers (`AWS::Credentials::Provider`, `get -> Keys`):
  - `static(access_key_id:, secret_access_key:, session_token: nil)`;
  - `env`: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`;
  - `profile(name)`: `AWS_PROFILE`, else `default`, from
    `~/.aws/credentials` (`AWS_SHARED_CREDENTIALS_FILE`) and
    `~/.aws/config` (`AWS_CONFIG_FILE`, `[profile name]`), the first
    file's settings winning. A profile with `role_arn` and
    `source_profile` is the role's temporary keys, from STS `AssumeRole`
    signed with the source profile's own keys (one level, or the profile
    itself; with `external_id`, `duration_seconds`, `role_session_name`
    when given), before the profile's own keys. With `credential_source`
    instead of `source_profile`, the role's keys are had with those of
    `env` (`Environment`), `instance` (`Ec2InstanceMetadata`) or
    `container` (`EcsContainer`); another value, or both settings
    (mutually exclusive, as in the SDKs), raises `AWS::ProfileError`
    before any request. A profile named by
    `AWS_PROFILE` (or given), or one with `role_arn` (the default one
    included), whose keys cannot be had raises `AWS::ProfileError`
    (`AWS::ProfileNotFound` when neither file has it), which stops a
    chain rather than find other keys;
  - `container`: ECS and EKS Pod Identity,
    `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` or `_FULL_URI` (https on any
    host; http only on loopback, `169.254.170.2`, `169.254.170.23` or
    `fd00:ec2::23`), with `AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE` or
    `AWS_CONTAINER_AUTHORIZATION_TOKEN`;
  - `instance`: EC2's metadata service with IMDSv2 (a token, then the
    role's keys), one attempt with 1-second timeouts;
    `AWS_EC2_METADATA_SERVICE_ENDPOINT`, `AWS_EC2_METADATA_DISABLED`;
  - `web_identity`: STS `AssumeRoleWithWebIdentity` (unsigned) with
    `AWS_WEB_IDENTITY_TOKEN_FILE` (read at each refresh), `AWS_ROLE_ARN`,
    `AWS_ROLE_SESSION_NAME`;
  - `chain(providers)`: the first that answers, tried in order until
    one does (`CredentialsError` names what each said). Once one has
    answered, it is the chain's: its later failure is the chain's error,
    never other keys of another identity. A profile named by
    `AWS_PROFILE`, or the default one when it has `role_arn`, that cannot
    give keys stops the chain (`AWS::ProfileError`);
  - `default`: `chain([env, web_identity, profile, container, instance])`.
- Temporary keys are cached and refreshed 5 minutes before they expire,
  by one task at a time (the others go on with the kept keys, or wait
  for the new ones); a failed refresh keeps the old keys until they
  expire, then raises `AWS::CredentialsError`. A refresh that failed, or
  brought keys already within the 5 minutes, is not tried again for 10
  seconds. Any error a
  provider meets is a `CredentialsError`; none follows redirects.
- `inspect` of a provider names it (`#<AWS::Credentials environment>`),
  never what it holds.
- STS is reached at `AWS_ENDPOINT_URL_STS` or `AWS_ENDPOINT_URL` when
  set, else its regional endpoint (`AWS_REGION`, `AWS_DEFAULT_REGION`,
  the profile's `region`), the global one without a region.
- For tests, the providers take `env:` (a hash in place of `ENV`) and
  `http:` (an `HTTP::Client`); `container` and `instance` take `base:`,
  the endpoint's base URL.
- `AWS::Profiles.region(name)`: a profile's region.

### Signature Version 4

```ruby
req = HTTP::Client::Request.post("https://sqs.ap-northeast-1.amazonaws.com/", body: form,
                                 content_type: "application/x-www-form-urlencoded")
signed = AWS::SigV4.sign(req, service: "sqs", region: "ap-northeast-1", credentials: creds.get)
url = AWS::SigV4.presign(HTTP::Client::Request.get("https://bucket.s3.amazonaws.com/key"),
                         service: "s3", region: "us-east-1", credentials: creds.get, expires_in: 15.minutes)
signer = AWS::SigV4::Signer.new("kms", "us-east-1", creds)   # signer.sign(req)
```

- `sign` adds `Host` (from the URL, when absent), `X-Amz-Date`,
  `X-Amz-Security-Token` (temporary keys), `X-Amz-Content-Sha256` (for
  `s3`, `content_sha256: true`, `payload_hash:` or `unsigned_payload: true`)
  and `Authorization`. Every header present is signed but
  `Authorization`, `User-Agent`, `Expect` and the hop-by-hop ones; a
  `Content-Length` the request carries is signed, and must be the body's
  size. The payload's hash is the body's SHA-256; a streamed body needs
  `payload_hash:` or `unsigned_payload: true`.
- Paths are normalized and each segment encoded (an encoded path twice)
  for every service but S3, whose paths are signed as they are.
- `presign` signs in the query (`X-Amz-Algorithm`, `-Credential`,
  `-Date`, `-Expires`, `-SignedHeaders`, `-Security-Token`, `-Signature`).
- `SigV4.parts` gives each stage (canonical request, string to sign,
  signature).

Tested against the official SigV4 test suite (`test/fixtures/sigv4/`,
from awslabs/aws-c-auth, Apache-2.0) at every stage, the S3
documentation's examples, and fake endpoints for every provider.

### Configuration

```ruby
config = AWS::Config.new(region: "ap-northeast-1", credentials: AWS::Credentials.default)
config = AWS::Config.default   # from the environment
```

`AWS::Config(region:, credentials:, endpoint: nil, http: HTTP::Client.default, retries: 3, timeout: 30.seconds, retry_base: 100.milliseconds, retry_cap: 20.seconds, service_endpoints: {})`:
the region, the credentials provider, an endpoint for every service
(a local emulator) or one per service, the HTTP client, the retries
(full-jitter backoff up to `min(retry_cap, retry_base * 2**n)`) and the
longest an attempt may take. `Config.default` reads `AWS_REGION`,
`AWS_DEFAULT_REGION` or the profile's `region`, `Credentials.default`,
`AWS_ENDPOINT_URL` and `AWS_ENDPOINT_URL_<SERVICE>` (`AWS_ENDPOINT_URL_KMS`).

### KMS

```ruby
kms = AWS::KMS.new(config)   # Shareable: one per program

out = kms.encrypt(key_id: "alias/app", plaintext: secret, encryption_context: {"tenant" => "t1"})
kms.decrypt(ciphertext_blob: out.ciphertext_blob, encryption_context: {"tenant" => "t1"}).plaintext

dk = kms.generate_data_key(key_id: "alias/app", key_spec: :aes_256)   # dk.plaintext, dk.ciphertext_blob
sig = kms.sign(key_id: "alias/sign", message: bytes, signing_algorithm: :rsassa_pkcs1_v1_5_sha_256).signature
kms.verify(key_id: "alias/sign", message: bytes, signature: sig, signing_algorithm: :rsassa_pkcs1_v1_5_sha_256)   # => Bool
pub = kms.get_public_key(key_id: "alias/sign")                 # pub.public_key: DER, Crypto::RSA::PublicKey.from_der
kms.describe_key(key_id: "alias/app").enabled?
kms.generate_random(number_of_bytes: 32).plaintext

box = AWS::KMS::Envelope.new(kms, "alias/app", {"tenant" => "t1"})
sealed = box.seal(large_secret)   # Bytes: a data key from KMS, AES-256-GCM
box.open(sealed)
```

- KMS's JSON 1.1 protocol (`POST /`, `X-Amz-Target: TrentService.<Operation>`),
  signed with SigV4. The operations: `encrypt`, `decrypt`,
  `generate_data_key`, `generate_data_key_without_plaintext`, `sign`,
  `verify` (`false` for `KMSInvalidSignatureException`),
  `get_public_key`, `describe_key`, `generate_random`; keyword
  arguments in snake_case, enumerations as symbols (`:aes_256`,
  `:rsassa_pss_sha_256`, `message_type: :digest`), answers as `data`
  values with `Bytes` for the base64 fields (`inspect` leaves
  plaintexts out).
- Throttling (`ThrottlingException`, 429), 5xx answers,
  `KMSInternalException`, `DependencyTimeoutException` and transport
  errors are tried again (every operation here is safe to send twice);
  an answer that may mean a clock skew (`InvalidSignatureException`,
  `SignatureDoesNotMatch`, `RequestExpired`, `RequestInTheFuture`,
  `RequestTimeTooSkewed`) with a `Date` over 4 minutes from ours is tried
  once more with the server's time, which later requests keep.
- Requests follow no redirect (`Config.client`, the default): a
  redirect would take the body (a plaintext) elsewhere. The region must
  be a host label.
- `KMS::Envelope.new(kms, key_id, encryption_context, allowed_keys =
  [])`: each `seal` makes a data key (`generate_data_key`, bound to the
  encryption context) and encrypts with AES-256-GCM. `open` decrypts the
  data key through KMS under the key ARN the sealed value names, with
  the same encryption context, but only when that ARN is the key
  `key_id` names (an alias or a bare id resolved with `describe_key`,
  asked again before refusing when it was kept, so that a moved alias is
  followed) or one of `allowed_keys` (key ARNs: an alias's previous key,
  for values sealed before it moved). Nothing else is ever passed to
  Decrypt. Why: a sealed value names its key, and the encryption context
  is no secret, so anyone could seal a value under a key of their own
  that the caller may decrypt with (another account's key whose policy
  allows the caller's role, with `kms:Decrypt` on `*` in the caller's
  IAM policy, or any key of the caller's own account it may decrypt
  with); KMS would give back the forger's data key, and the value would
  open to a plaintext of their choosing. The AWS Encryption SDK guards
  against the same with its strict mode. The key an alias or a bare id
  resolved to is kept 5 minutes; with an alias or a bare id, `open`
  needs `kms:DescribeKey` too, and each value refused costs one
  DescribeKey call (KMS's request quota is shared). The sealed bytes,
  for other code to read:

  | bytes | |
  |---|---|
  | 1 | version: `1` |
  | 1 | the length `a` of the KMS key's ARN (up to 255) |
  | `a` | the ARN (UTF-8), as `GenerateDataKey` answered it |
  | 2 | the length `n` of the encrypted data key, big-endian |
  | `n` | the encrypted data key (KMS's `CiphertextBlob`) |
  | 12 | the AES-GCM nonce |
  | rest | the ciphertext, then its 16-byte tag |

  The data key is 32 bytes (AES-256); the bytes before the ciphertext
  are its additional authenticated data.

### Errors

- `AWS::Error`, the root of the errors this cog raises;
  `AWS::CredentialsError`, no credentials could be had (the message
  says what was tried, never a key); `AWS::ServiceError`, a service's
  error: `code` (`NotFoundException`, the service's prefix dropped:
  match on it), `detail` (its message), `status`, `request_id`;
  `AWS::TransportError`, a request that could not be made or answered
  (the `HTTP::Client::Error`'s class in the message). No message holds
  a request's body or keys.

### Not there yet

Other services (S3, SQS, STS beyond the credentials), presigned S3
uploads beyond `SigV4.presign` itself, a data-key cache for
`KMS::Envelope`.

### Tests

`quarz test` runs against fake endpoints (a fake KMS in Quarz that
recomputes each request's signature) and temporary files. An opt-in
test against a real KMS reads `AWS_TEST_KMS_KEY_ID` (a symmetric key
the environment's credentials may use; `AWS_ENDPOINT_URL_KMS` for a
local emulator) and is never run in CI.

## License

MIT No Attribution (LICENSE).
