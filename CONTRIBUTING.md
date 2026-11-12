# Contributing to Doorbridge

Thank you for your interest. The project is in an early design phase; most work currently happens in the issues.

## Developer Certificate of Origin (DCO)

Every commit must be signed off. By adding a `Signed-off-by` line you certify the [Developer Certificate of Origin 1.1](https://developercertificate.org/): that you wrote the change or otherwise have the right to submit it under the project's licence.

```bash
git commit -s -m "feat(bridge): add discovery response"
```

This adds a line like:

```
Signed-off-by: Jane Doe <jane@example.com>
```

Pull requests with commits that lack a valid sign-off cannot be merged. A DCO check will run on every pull request.

## Commit messages

Use [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/):

```
<type>(<optional scope>): <description>
```

Common types: `feat`, `fix`, `docs`, `chore`, `ci`, `refactor`, `test`, `perf`. Mark breaking changes with `!` or a `BREAKING CHANGE:` footer. Releases and the changelog are generated from these messages.

## No real environment data

Never commit real hostnames, domains, IP addresses, names, serial numbers, account IDs or tokens, not even in issues, logs or examples. Use placeholders such as `bridge.example.com`, `192.0.2.10` or `front-door`.

## Licence

By contributing you agree that your contributions are licensed under the [Apache License 2.0](LICENSE).
