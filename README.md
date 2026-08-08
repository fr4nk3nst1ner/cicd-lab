# cicd-lab

A small teaching lab that shows how a GitHub Actions workflow pivots into AWS with OIDC, and how to scope the IAM trust so nothing but this repo's `main` branch can do it. The repo is public on purpose. The security lives in the AWS trust policy, not in hiding the code.

## What it shows

GitHub Actions can mint a short lived OIDC token and exchange it for AWS credentials with no stored secrets. Whether AWS actually hands those credentials over depends entirely on the IAM role's trust policy, not on who triggered the workflow.

## Why a fork or a random PR cannot reach the AWS account

The role trust is pinned to exactly one subject:

```
repo:fr4nk3nst1ner/cicd-lab:ref:refs/heads/main
```

- A fork's `pull_request` run cannot obtain an OIDC token with write permission, so it never gets far enough to try.
- A pull request inside this repo runs with subject `...:pull_request`, which does not match `...:ref:refs/heads/main`, so AWS denies it.
- Only a push to `main`, which needs write access to this repo, produces a matching subject.

Open a pull request and watch the job fail at the credentials step. That failure is the control doing its job.

## The teaching contrast

- `trust-policies/hardened.json` is what is deployed. It pins the subject to `main`.
- `trust-policies/vulnerable.json` is the mistake. It wildcards the subject, which would let any branch or PR, including a fork's, assume the role. Never deploy that outside a throwaway account.

## Blast radius

The role can read exactly one thing, a fake secret in Parameter Store used as a capture the flag trophy. It cannot list buckets, read real data, or create anything. Even a successful pivot lands in a sandbox. Also note that Actions logs on a public repo are public, so the workflow is written to print only that fake trophy, never real account details.

## Files

- `.github/workflows/oidc-pivot.yml` the workflow
- `trust-policies/hardened.json` and `trust-policies/vulnerable.json` the good and bad trust policies
- `docs/ARCHITECTURE.md` the flow
