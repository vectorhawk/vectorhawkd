# vectorhawkd

The VectorHawk runner — a per-user local agent that delivers governed AI tools and skills to AI clients (Claude Code, Cursor, VS Code, Gemini CLI, and others) over the Model Context Protocol.

> **Internal name:** `vectorhawkd`. **Public/marketing name:** "the VectorHawk runner" or just "the runner". Use the public name in user-facing docs, error messages, and marketing copy.

## Architecture

The runner is a **per-user daemon** with a **thin per-session stdio shim**:

```
AI client  --stdio JSON-RPC-->  vectorhawk mcp serve  (shim)
                                          |
                                          | Unix socket / named pipe
                                          v
                                    vectorhawkd  (daemon, LaunchAgent / systemd-user / Win service)
                                          |
                                          v
                          Backend MCP servers, registry, gateway, OAuth, audit
```

The daemon (`vectorhawkd`) holds all cross-session state: backend MCP connections, audit buffer, policy cache, OAuth callbacks, credential broker client, session budgets. The shim relays MCP traffic between the AI client and the daemon, with an in-process fallback if the daemon is unreachable.

See [`../context/RUN1_DAEMON_PIVOT_DECISION.md`](../context/RUN1_DAEMON_PIVOT_DECISION.md) and [`../context/RUN1_DAEMON_SPEC.md`](../context/RUN1_DAEMON_SPEC.md) for the design rationale.

## Crate layout

```
crates/
├── vectorhawkd-manifest/   library — skill bundle types, manifest parsing
├── vectorhawkd-core/       library — state, installer, registry client, policy, executor
├── vectorhawkd-mcp/        library — MCP protocol, aggregator, backend trait
├── vectorhawkd-daemon/     binary — vectorhawkd long-running agent
├── vectorhawkd-shim/       binary — stdio↔socket relay with in-process fallback
└── vectorhawkd-cli/        binary — vectorhawk user CLI (skill, mcp, daemon, doctor commands)
```

## Binaries

- `vectorhawkd` — the daemon. Started by LaunchAgent / systemd-user / Windows service. Listens on a Unix domain socket / named pipe. Not directly invoked by users.
- `vectorhawk` — the user-facing CLI. `vectorhawk mcp serve` is what AI clients spawn (it's the shim). `vectorhawk skill install`, `vectorhawk doctor`, etc. are user commands.

## Build

```bash
cargo build
cargo build --release
cargo test
cargo clippy
```

## Status

Pre-alpha. M0 spike in progress. See [`../context/IMPLEMENTATION_PLAN_v3.md`](../context/IMPLEMENTATION_PLAN_v3.md) Phase RUN1.

## Releasing

Releases are cut by pushing a semver git tag. GitHub Actions builds release binaries for three targets
(`aarch64-apple-darwin`, `x86_64-apple-darwin`, `x86_64-unknown-linux-gnu`) and attaches them to the
matching GitHub Release automatically.

To cut a release:

```bash
git tag v0.1.0
git push origin v0.1.0
```

Each release attaches per-target tarballs named `vectorhawk-<version>-<triple>.tar.gz`, each containing
`vectorhawk`, `vectorhawkd`, `vectorhawkd-shim`, `LICENSE`, and `README.md`. A SHA256 checksum file
(`vectorhawk-<version>-<triple>.tar.gz.sha256`) is attached alongside each tarball.

Pre-release tags (e.g. `v0.1.0-rc.0`) are marked as pre-releases on GitHub. Stable tags produce
full releases.

D1.2 (curl-install script) and D1.3 (Homebrew tap) consume these release artifacts. The tarball name
and SHA256 format are the stable interface those scripts rely on.

## Uninstalling

- **curl install**: `rm -rf ~/.vectorhawk` and remove the `$PATH` entry the installer added to your
  shell rc file (e.g. `~/.zshrc` or `~/.bashrc`).
- **Homebrew**: `brew uninstall vectorhawk`

Neither of these uninstall the LaunchAgent or systemd user unit. To remove those, run
`vectorhawk daemon uninstall` before removing the binaries.

## License

Apache-2.0. See [LICENSE](./LICENSE).
