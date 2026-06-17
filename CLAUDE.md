# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This repository serves two roles:

1. **GitHub special profile repository.** Because the repo name (`next-nf/next-nf`) matches the owner's username, the contents of `README.md` are rendered on the owner's GitHub profile page.
2. **Claude Code plugin marketplace** for the `next-nf` org. It hosts a `.claude-plugin/marketplace.json` catalog and the plugins it ships, so contributors can install the org's shared AI skills.

There is still no application source code, build tooling, package manifest, or test suite here — nothing is compiled or executed.

## Layout

```
.
├── README.md                       # rendered on the GitHub profile
├── .claude-plugin/
│   └── marketplace.json            # marketplace catalog: name "next-nf", lists plugins
└── plugins/
    ├── nf-docs/                    # org-wide documentation-standard plugin
    │   ├── .claude-plugin/plugin.json
    │   └── skills/
    │       └── documenting-network-functions/
    │           ├── SKILL.md                 # operational summary (auto-invoked)
    │           ├── documentation-style.md   # authoritative ETSI-derived standard
    │           └── templates/               # copy-in doc templates (config, interface,
    │                                         #   runbook, troubleshooting, diagrams)
    └── nf-conventions/             # org-wide architecture/design/conventions plugin
        ├── .claude-plugin/plugin.json
        └── skills/
            └── nf-architecture/
                ├── SKILL.md                 # summary + index + set policies
                └── reference/               # progressive-disclosure guides, loaded on demand:
                                             #   architecture (umbrella + sbi/api/web naming),
                                             #   erlang-otp (OTP-29 + native records),
                                             #   diameter, observability (OTEL),
                                             #   interfaces-map, conventions, deployment-philosophy
```

## Working in this repo

- **Profile content** lives in `README.md` (GitHub-flavored markdown). HTML comments (`<!-- ... -->`) are hidden from the rendered profile.
- **Marketplace catalog:** `.claude-plugin/marketplace.json`. Each entry's `source` is a path under `plugins/` (relative-path sources resolve because this marketplace is git-based). Add a plugin by creating its directory under `plugins/` and adding a catalog entry.
- **Plugins** carry their manifest in `.claude-plugin/plugin.json`; bundled `skills/`, `commands/`, `agents/`, `hooks/` directories sit at the plugin root, not inside `.claude-plugin/`.
- **The `nf-docs` skill is org-wide**, not specific to this repo: it defines the operator-documentation standard for every next-nf network function (SMF/PGW, UDR/HSS, PCF/PCRF, CHF). It was generalized from the `documenting-hss` skill in the `udr` repo. Keep its terminology and examples spanning all the network functions, not narrowed to one.
- **The `nf-conventions` skill is org-wide**: shared architecture, design rationale, and engineering conventions for all four components. Its `reference/` files state the **target convention** and link to the **tracking epic**; per-repo state and migration actions live in GitHub issues (see `reference/issue-tracking.md`). `udr` is the reference implementation for Diameter, observability, and native records. When editing these files, keep them org-wide — do not narrow to one component, and do not add per-repo state tables (file issues instead).

## Validation

There is no build or test step. Validate manifests after editing them:

```sh
claude plugin validate .claude-plugin/marketplace.json   # catalog
claude plugin validate plugins/nf-docs                    # plugin manifest
claude plugin validate plugins/nf-conventions             # plugin manifest
```

## Installing the marketplace (for reference)

```
/plugin marketplace add next-nf/next-nf
/plugin install nf-docs@next-nf
```
