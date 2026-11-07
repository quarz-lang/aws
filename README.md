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
    when given), before the profile's own keys. A profile named by
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

### Errors

- `AWS::Error`, the root of the errors this cog raises;
  `AWS::CredentialsError`, no credentials could be had (the message
  says what was tried, never a key).

## License

MIT No Attribution (LICENSE).
