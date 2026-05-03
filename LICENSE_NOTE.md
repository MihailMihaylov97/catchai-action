# Licensing — wrapper vs. binary

This repository contains **only the GitHub Action wrapper** — `action.yml`,
docs, smoke tests. Everything in this repo is MIT-licensed (see
[`LICENSE`](LICENSE)) and may be freely forked, modified, or
redistributed.

The Action **does not contain the catchai source code**. When the
Action runs, it downloads the catchai binary from a separate distribution
channel:

```
https://raw.githubusercontent.com/MihailMihaylov97/catchai-dist/main/install.sh
```

The catchai binary itself is governed by its own terms, set by its
copyright holder. Installing or running catchai through this Action
constitutes acceptance of those terms.

This split mirrors the standard pattern used by other commercial
security tools that ship via GitHub Actions:

* `aquasecurity/trivy-action` — open-source Action wrapper, separate
  binary licensing
* `snyk/actions/python` — open-source Action wrapper, Snyk binary
  governed by Snyk's terms
* `mongodb/mongo-tools` — open-source Action wrapper, MongoDB tools
  under their own license

The MIT license on this repository covers ONLY the YAML wrapper, the
PR-comment script, the smoke workflow, and the docs. It does not
extend to the catchai binary that the Action installs at runtime.
