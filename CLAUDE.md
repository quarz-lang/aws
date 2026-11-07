# Working on aws

A cog of Quarz (github.com/quarz-lang/quarz): its rules apply here.
Code, comments, commit messages and the docs are in English.

## Layout

- `Cog.toml` (`module = "AWS"`), `src/` (declarations only), `test/*_test.qz`
  (`require "test"`, Test::Case), README.md, LICENSE.
- Written in Quarz over its standard library only (`http`, `uri`, `json`,
  `digest`, `base64`, `crypto`, `time`): AWS credentials, Signature
  Version 4, and service clients as modules (`AWS::KMS`). The design and
  its decisions are in issue #1.

## Before committing

```sh
quarz check
quarz test              # interpreted
quarz test --release    # compiled
quarz test --doc        # the doc comments' examples
quarz doc --check
```

Use a `quarz` built from Quarz's main (`cargo build --release` there).
The tests run against fake endpoints (`HTTP::Server` on port 0) and
temporary files. The opt-in test against a real KMS reads
`AWS_TEST_KMS_KEY_ID` and real credentials from the environment
(`AWS_ENDPOINT_URL_KMS` for a local emulator), and is never run in CI.

- Work on a branch and open a pull request (below); stage files by explicit path.
- Review the branch's diff with a fresh subagent before the pull request
  leaves draft (commits on a draft are fine without it), and fix what it
  finds that needs no decision.
- Never print secrets (access keys, session tokens, plaintexts); tests use
  the AWS documentation's example keys.
- Do not put a `Claude-Session:` line in commit messages, nor a Claude
  session URL in pull request descriptions.

## Pull requests

- Open a draft pull request as soon as the branch has its first commit,
  while the local checks run: CI then runs in parallel with `quarz test`.
- Force-pushing after the subagent review or the local checks changed
  the commits is fine at any stage (amend or rebase as needed, onto
  main too): the merge names the head commit it was approved on.
- Mark it ready for review only when the subagent review of the branch's
  diff and every local check above have passed; from then on
  review every change, added or rewritten, before pushing it.
- The description says what was verified.

## Issues

- Open work lives in GitHub issues of the repository that owns the code,
  one per defect or change. The title is a sentence in the commit-message
  style, saying what is wrong or what is wanted.
- The body has the headings **Symptom**, **Reproducer**, **Cause** (when
  known; say when it is a guess), **Direction** (`Decided (date): ...`,
  or the options with a recommendation) and **Found in** (the pull
  request or review that found it).
- Labels: `bug`, `enhancement`, `documentation`; `soundness` when a
  guarantee of the type or ownership system breaks (the highest
  priority); `performance`; `design` when a language or API decision is
  needed; `needs-decision` while it waits for the maintainer;
  `after-0.1.0`.
- A bug found while implementing or reviewing that is outside the pull
  request's scope is filed as an issue that says which pull request
  found it, not fixed in that pull request nor left in a comment.
- A pull request that fixes an issue says `Closes #N` in its description.
  A decision taken about an issue is recorded as a comment on it.
