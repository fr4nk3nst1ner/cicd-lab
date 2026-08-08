# cicd-lab

Shows how a GitHub Actions workflow reaches AWS with OIDC, and how to scope the IAM trust so only this repo's main branch can do it. The repo is public on purpose. The security is in the trust policy, not in hiding the code.

## The idea

A workflow asks GitHub for a short lived OIDC token and trades it for AWS credentials. No stored keys. AWS decides whether to hand over credentials by matching the token's `sub` claim against the role's trust policy. That match is the only thing that matters. Who triggered the run and whether the repo is public are irrelevant.

## Why a fork or a PR cannot reach the account

The role trusts one subject:

```
repo:fr4nk3nst1ner@28953657/cicd-lab@1328186569:ref:refs/heads/main
```

The `@28953657` and `@1328186569` are the owner and repo IDs. This account issues OIDC subjects with those IDs, which also blocks anyone from deleting the repo and re-registering the name.

- A fork PR never gets `id-token: write`, so it cannot mint a token to try.
- A PR in this repo gets subject `...:pull_request`, which does not match `...:ref:refs/heads/main`, so AWS denies it.
- Only a push to main, which needs write access, produces a matching subject.

Open a PR and the credentials step fails. That is the control working.

## What is deployed vs what is a mistake

- `trust-policies/hardened.json` is live. It pins the subject to main.
- `trust-policies/vulnerable.json` wildcards the subject. That version would let any branch or PR assume the role. Never run it outside a throwaway account.

## Blast radius

The role can read one fake secret in Parameter Store and nothing else. No bucket listing, no real data, no writes. The workflow prints only that fake value, so the public Actions log stays clean.

## Files

- `.github/workflows/oidc-pivot.yml`
- `trust-policies/hardened.json`, `trust-policies/vulnerable.json`
- `docs/ARCHITECTURE.md`
