# clawker-bundle-example

A working example [clawker](https://github.com/schmitthub/clawker) bundle —
fork this repo to build your own. It ships four harnesses and eight
stacks:

| Component | Address | What it does |
|-----------|---------|--------------|
| `harnesses/claude` | `schmitthub.test-bundle.claude` | [Claude Code](https://code.claude.com) via the canonical installer (version resolved from npm `@anthropic-ai/claude-code`). Declares the `node` stack, persists `~/.claude` as a volume, seeds statusline/settings on first boot, stages host `~/.claude` state (settings allowlist, agents, skills, commands, plugin registry with path rewrites), and floors egress at the Anthropic API/OAuth/telemetry domains |
| `harnesses/codex` | `schmitthub.test-bundle.codex` | [Codex CLI](https://github.com/openai/codex) via the canonical installer (version resolved from GitHub release tags `rust-v*`). Self-contained binary — no stack. Persists `~/.codex` + `~/.agents` as volumes, stages host global `AGENTS.md` + prompts + `~/.agents`, installs `/etc/codex/config.toml` with the managed prompt as developer instructions, and floors egress at the OpenAI API/auth domains with `chatgpt.com` path-scoped to the codex backend |
| `harnesses/opencode` | `schmitthub.test-bundle.opencode` | [OpenCode](https://opencode.ai) via the canonical installer (version resolved from npm `opencode-ai`). Persists `~/.config/opencode` + `~/.local/share/opencode` as volumes, stages the host's global `AGENTS.md`, and floors egress at `models.dev` + `api.anthropic.com` |
| `harnesses/openclaw` | `schmitthub.test-bundle.openclaw` | [OpenClaw](https://openclaw.ai) gateway via npm (version resolved from npm `openclaw`). The one **daemon** harness — its CMD is a long-lived service, not an interactive CLI. Installs root-global so the exec-form CMD resolves, bakes Homebrew so brew-backed skills stay visible, persists `~/.openclaw` + `~/.config/openclaw` + `~/.local` + `~/.cache` as volumes, and floors egress at the model providers, the plugin registry, and the package registries runtime skill installs reach for |

The `claude` and `codex` harnesses are patterned on clawker's embedded
harnesses — complete worked examples of every harness surface (stacks,
volumes, seeds + assets, staging with `json_keys`/`json_rewrites`,
path-scoped egress, both npm and github-release version resolvers). The
`opencode` harness shows the minimal shape.

The `openclaw` harness is the odd one out, and deliberately so: it shows
what a harness looks like when the thing being harnessed is a **service**
rather than a CLI you sit in front of. Its CMD never exits, it is reached
over a published port instead of a terminal, and it carries a `HEALTHCHECK`.
Run it detached:

```bash
clawker run --detach -p 127.0.0.1:18789:18789 @:schmitthub.test-bundle.openclaw
```

It also demonstrates two things worth stealing for any harness whose CMD is
a daemon. First, exec-form CMD is resolved by `clawkerd` against the image
`PATH` alone — no shell runs, so `.zshenv` and nvm's user-owned prefix are
invisible to it; installing root-global is what keeps the binary findable.
Second, a harness can pre-create a directory in its root block to fix
ownership of a volume's *parent* before the engine's volume block runs.

The `stacks/` directory ships eight example stacks — `go`, `node`,
`python`, `rust`, `java`, `ruby`, `cpp`, and `dotnet`, addressed as
`schmitthub.test-bundle.<name>` — patterned on clawker's embedded ones.
Together they demonstrate every stack pattern: root-scope and user-scope
fragments (`node` ships both), checksum-verified official tarballs (`go`),
GPG-verified installs (`node`), `curl | sh` installer scripts (`python`,
`rust`, `dotnet`), plain apt toolchains (`java`, `ruby`, `cpp`), shared
world-writable state dirs (`go`'s GOPATH, `ruby`'s GEM_HOME), version-pin
ARGs over floating channels, and the self-guard convention (every fragment
skips itself when the image already provides the tool).

Declare and build:

```yaml
bundles:
  - url: https://github.com/schmitthub/clawker-bundle-example.git
    ref: v2026.7.1
```

```bash
clawker bundle install
clawker build -t schmitthub.test-bundle.codex
```

## Anatomy of a harness

```
harnesses/<name>/
├── harness.yaml             # manifest: version resolver, stacks, volumes,
│                            # seeds, staging, managed_prompt, egress floor
├── Dockerfile.harness.tmpl  # template blocks composed into clawker's
│                            # master Dockerfile (install steps, ENV, CMD)
└── assets/                  # optional files staged into the build context
                             # (referenced by seeds: and COPY)
```

## Anatomy of a stack

```
stacks/<name>/
├── stack.yaml                  # manifest: description
├── Dockerfile.stack-root.tmpl  # root-scope install steps (system-wide
│                               # toolchains), rendered into the base image
└── Dockerfile.stack-user.tmpl  # user-scope install steps (per-user version
                                # managers like nvm/rustup) — ship one
                                # fragment or both
```

Fragments are Go templates with one variable, `{{.BuildKitEnabled}}`, for
emitting `--mount=type=cache` apt cache mounts only when BuildKit renders
the build. Every fragment self-guards: it skips its install when the image
already provides the tool, so declaring a stack that the base already
carries is always safe.

Planned additions (ongoing): a `pi` harness and the remaining embedded
components.
