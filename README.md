# trust-lockfile-transitive-peer-deps-v11

Patterns exercised: `trust-lockfile` + `transitive-peer-deps-v11`

pnpm version under test: 11.3

## Feature exercised

This probe exercises two pnpm 11.3 features that affect Mend
SCA detection:

1. `trustLockfile: true` — set via `.npmrc` as
   `trust-lockfile=true`. Instructs `pnpm install` to skip
   supply-chain integrity verification and use the lockfile
   as-is. Relevant to the `install_command` category because
   it changes how the pnpm CLI behaves when Mend's pre-step
   runs `pnpm install` (configured via `npm.runPreStep=true`
   in `whitesource.config`).

2. Transitive peer dependency resolution changes (pnpm 11.3)
   — the lockfile uses v9 format with peer-decorated snapshot
   keys in the `snapshots` section. Under pnpm 11.3 the
   algorithm that deduplicates peer dep instances was revised,
   so snapshot keys carry full peer decoration strings such as:
   - `react-dom@18.2.0(react@18.2.0)`
   - `@testing-library/react@14.3.1(react@18.2.0)(react-dom@18.2.0(react@18.2.0))`

   Mend's `PnpmLockCollector` (via `PnpmParserV9Impl`) must
   strip these decorations to recover bare package names and
   versions. Failure to do so produces missing or duplicated
   entries in the detected tree.

## Dependency graph

```
root
├── react@18.2.0                         (direct, registry)
│   └── loose-envify@1.4.0               (transitive, registry)
│       └── js-tokens@4.0.0              (transitive, registry)
├── react-dom@18.2.0                     (direct, registry)
│   ├── react@18.2.0                     (peer, already listed)
│   └── scheduler@0.23.2                 (transitive, registry)
│       └── loose-envify@1.4.0           (transitive, already listed)
└── @testing-library/react@14.3.1        (direct, registry)
    ├── react@18.2.0                     (peer, already listed)
    └── react-dom@18.2.0                 (peer, already listed)
```

## Key pnpm 11.3 lockfile details

The `snapshots` section uses peer-decorated keys:

```yaml
snapshots:
  react-dom@18.2.0(react@18.2.0):
    dependencies:
      react: 18.2.0
      scheduler: 0.23.2
  '@testing-library/react@14.3.1(react@18.2.0)(react-dom@18.2.0(react@18.2.0))':
    dependencies:
      react: 18.2.0
      react-dom: 18.2.0(react@18.2.0)
```

These decorated keys represent the pnpm 11.3 transitive peer
dep resolution behavior. Mend must parse the `(peer@version)`
suffixes and strip them to identify the underlying package.

## Expected dependency tree

All packages that should appear in `expected-tree.json`:

| Package | Version | Source | Direct? |
|---|---|---|---|
| `react` | 18.2.0 | registry | yes |
| `react-dom` | 18.2.0 | registry | yes |
| `@testing-library/react` | 14.3.1 | registry | yes |
| `loose-envify` | 1.4.0 | registry | no |
| `js-tokens` | 4.0.0 | registry | no |
| `scheduler` | 0.23.2 | registry | no |

The `trustLockfile` setting does NOT change the expected tree —
Mend reads `pnpm-lock.yaml` directly and is not affected by the
pnpm CLI flag.

## Mend failure modes targeted

### trust-lockfile category
- Mend incorrectly interprets `trust-lockfile=true` in `.npmrc`
  and skips integrity verification of packages, leading to a
  truncated or incorrect tree.
- Pre-step (`npm.runPreStep=true`) runs `pnpm install` with the
  trust flag active; Mend should still see the complete tree
  from the resulting `node_modules/`.

### transitive-peer-deps-v11 category
- Snapshot keys like `react-dom@18.2.0(react@18.2.0)` not
  parsed → `react-dom` missing from detected tree.
- Long compound decoration
  `@testing-library/react@14.3.1(react@18.2.0)(react-dom@18.2.0(react@18.2.0))`
  causes parser to bail → `@testing-library/react` absent.
- Duplicate entries: one for the bare key (from `packages`
  section) and one for the snapshot key (from `snapshots`
  section).
- Peer dep resolved at wrong level (root vs sub-package).

## Mend config

**Bucket A** — `.whitesource` pins `pnpm: ">=11.3 <12"` and
`node: ">=20 <21"`. pnpm has no dynamic version detection from
the manifest, so every probe must pin the toolchain to prevent
non-reproducible transitive sets across Mend scan runs.

Version range `>=11.3 <12` is used (rather than the default
`>=9 <10`) because this probe specifically targets pnpm 11.3
behavior. The peer-decorated snapshot keys and `trustLockfile`
setting are pnpm 11.3 features.

**Additional dimensions:**

- `whitesource.config` included in probe root with
  `npm.runPreStep=true` and `npm.ignoreScripts=true`.
  This exercises the install_command path under `trustLockfile`
  — the pre-step runs `pnpm install` with trust flag active,
  so integrity errors are silently swallowed rather than
  blocking install, but the resulting `node_modules/` still
  matches the lockfile and Mend reports the full tree.

## Probe metadata

| Field | Value |
|---|---|
| pm | pnpm |
| pm_version_tested | 11.3 |
| pm_version_under_test | 11.3 |
| patterns | trust-lockfile, transitive-peer-deps-v11 |
| categories | install_command, tree_structure |
| lockfile_version | 9.0 |
| schema_version | 1.2 |
| generated_at | 2026-05-24T09:06:20Z |
| resolver_sha | 938be864c47ba3f0f009464599ab689c2876f716 |
| resolver_fetched_at | 2026-05-24T09:05:49+00:00 |
