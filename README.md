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

- `AWS::Error`, the root of the errors this cog raises.

## License

MIT No Attribution (LICENSE).
