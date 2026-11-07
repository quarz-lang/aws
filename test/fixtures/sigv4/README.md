Cases of the AWS Signature Version 4 test suite, copied unmodified from
[awslabs/aws-c-auth](https://github.com/awslabs/aws-c-auth)
`tests/aws-signing-test-suite/v4/` at commit `bdb7cef99d8bd1ad3c189b1400140a7787e47105`. They are
licensed under the Apache License 2.0 (`LICENSE`, `NOTICE` here),
not under this repository's license.

Each directory is one case: `context.json` (the keys, region, service,
time and options), `request.txt` (the request), and the expected
canonical request, string to sign, signature and signed request, for
the `Authorization` header (`header-*`) and for the query
(`query-*`, `AWS::SigV4.presign`). `test/sigv4_test.qz` runs every
case here through `AWS::SigV4` and compares each stage.

The suite's other cases are left out:

- `get-header-value-multiline`: a header value with a line break, which
  `HTTP::Headers` refuses (a control character).
- `get-space-normalized`, `get-space-unnormalized`: a space in the
  path, which `HTTP::Client::Request` refuses in a URL.
- `*-unnormalized`: a service other than S3 signed without normalizing
  the path; as the SDKs, `AWS::SigV4` normalizes for every service but
  S3.
- `post-sts-header-after`: the session token added after signing
  (`omit_session_token`), which `AWS::SigV4` does not do.
