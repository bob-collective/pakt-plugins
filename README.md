# Pakt plugins

Pakt agent plugin v0.1.4 for Claude Code and Codex. Authentication starts
when Pakt is first used; never share a wallet seed or private key.

```sh
claude plugin marketplace add bob-collective/pakt-plugins
claude plugin install pakt@bob-collective
codex plugin marketplace add bob-collective/pakt-plugins --ref main
codex plugin add pakt@bob-collective
```

Update with `claude plugin update pakt@bob-collective` or
`codex plugin marketplace upgrade bob-collective`.
