# Codex Sol/Astra Orchestrator + Luna Subagents

Install a Codex profile with GPT-6 Astra, GPT-6.1 Sol, or GPT-6 Luna as the orchestrator,
GPT-6 Luna execution subagents, and an independent reviewer.

The Sol profiles use `gpt-6.1-sol` for both the orchestrator and reviewer.

## Orchestration topology

The diagram shows the Astra (Pro) and Sol orchestrators. Plus uses a Luna
root, as shown in the profile table below. Execution roles use GPT-6 Luna;
the reviewer uses Astra for Pro/Plus and Sol for Sol profiles.

```text
           GPT-6 Astra / GPT-6.1 Sol
             root / orchestrator
                      |
      +---------------+---------------+
      |               |               |
   explorer          worker         researcher
  GPT-6 Luna       GPT-6 Luna       GPT-6 Luna
      |               |
      +-------+-------+
              |
           tester
         GPT-6 Luna
              |
          reviewer
      GPT-6 Astra / GPT-6.1 Sol
              |
              v
           root agent
      integrate + verify
```

## Setup

1. Clone this repository and enter it:

   ```sh
   git clone https://github.com/donvito/codex-astra-luna-orchestrator.git
   cd codex-astra-luna-orchestrator
   ```

2. Run the installer for your platform:

   macOS/Linux:

   ```sh
   ./setup.sh
   ```

   Windows PowerShell:

   ```powershell
   powershell -ExecutionPolicy Bypass -File .\setup.ps1
   ```

   PowerShell 7:

   ```powershell
   pwsh -File .\setup.ps1
   ```

3. When prompted, enter an **existing target repository other than this one**,
   choose a profile by number or name (Enter selects Pro), and confirm which
   components to install. For example:

   ```text
   Target repository path: ../my-project
   Select Profile [1-6] (default 1): 5
   ```

The installer copies the selected configuration to `.codex/`, its skill to
`.agents/`, and project instructions to `AGENTS.md`. Existing component files
are updated only after confirmation; existing `AGENTS.md` content is preserved.
Codex loads project-scoped configuration only for trusted projects.

### Installed target project

If you install all three components into `../my-project`, the installer adds
these paths alongside the project's existing files:

```text
my-project/
├── .codex/
│   ├── config.toml
│   └── agents/
│       ├── explorer.toml
│       ├── researcher.toml
│       ├── reviewer.toml
│       ├── tester.toml
│       └── worker.toml
├── .agents/
│   └── skills/
│       └── astra-orchestrator/
│           └── SKILL.md
└── AGENTS.md
```

`profiles/<profile>/codex/` becomes `.codex/`, and
`profiles/<profile>/agents/` becomes `.agents/`. The root `AGENTS.md` is
copied to the target, or its instructions are appended if that file exists.

## How to use the skill

From the target repository, launch Codex CLI. For the example above:

```sh
cd ../my-project
codex
```

For complex work, Codex may select the skill automatically, or you can invoke
it explicitly:

```text
$astra-orchestrator
Implement the invoice export endpoint. Use the explorer to map the path,
a worker to implement it, and the tester and reviewer to verify it.
```

The skill keeps the `astra-orchestrator` name in every profile so the shared
`AGENTS.md` works; the Sol profiles use Sol according to their configuration.

## Profiles

| Choice | Profile | Root | Execution roles | Reviewer | Concurrent subagents |
|---|---|---|---|---|---:|
| 1 (default) | `pro` | Astra medium | Luna max | Astra low | 4 |
| 2 | `plus` | Luna max | Luna medium | Astra low | 4 |
| 3 | `pro-max-2-subagents` | Astra medium | Luna max | Astra low | 2 |
| 4 | `plus-max-2-subagents` | Luna max | Luna medium | Astra low | 2 |
| 5 | `GPT6-SolMax-LunaMax` | Sol max | Luna max | Sol max | 4 |
| 6 | `GPT6-SolMedium-LunaMax` | Sol medium | Luna max | Sol medium | 4 |

All models above are GPT-6. Execution roles are explorer, worker, tester,
and researcher; named roles pin their models and reasoning levels independently
of the default subagent settings. Each ready-to-copy profile lives under
`profiles/<profile>/`.

For manual project setup, copy the selected profile's `codex/` and `agents/`
to the target repository as `.codex/` and `.agents/`, and add this repository's
`AGENTS.md`. For personal/global setup, copy its `codex/agents/` into
`~/.codex/agents/`, its `agents/skills/astra-orchestrator/` into
`~/.agents/skills/`, and **merge**, rather than replace, its `codex/config.toml`
settings into `~/.codex/config.toml`. Do not overwrite other existing Codex
settings.

## Model context windows

All six profiles include a commented, opt-in 1M-class client configuration:

```toml
model_context_window = 1_050_000
model_auto_compact_token_limit = 900_000
model_auto_compact_token_limit_scope = "total"
model_catalog_json = "/absolute/path/to/models-1m.json"
```

Leave this block disabled unless your provider confirms that **every selected
root, execution, and reviewer model** supports that window for your account.
These settings and a local catalog cannot enable backend support. Upstream now
uses `gpt-6-astra`, `gpt-6-luna`, and `gpt-6.1-sol`; older model names are not
substitutes for checking those exact models.

Compatibility was checked with Codex CLI `0.159.0-alpha.3` and the current
[configuration schema](https://github.com/openai/codex/blob/afb436df8b70bb5bc57b86d9a3e829968988cd21/codex-rs/core/config.schema.json).
All four keys are supported. Codex caps `model_context_window` at the selected
model's `max_context_window` metadata. In the tested binary's bundled catalog,
Astra and Luna have a 272,000 default and an 872,000 maximum; `gpt-6.1-sol` is
absent. These are client metadata observations, not verified backend limits.
The original blanket claim of 1,050,000-token model support is therefore not
established by this check.

First inspect your actual catalog (requires `jq` for these examples):

```sh
codex --version
codex debug models > /tmp/codex-models.json
jq '.models[] | select(.slug == "gpt-6-astra"
  or .slug == "gpt-6-luna" or .slug == "gpt-6.1-sol")
  | {slug, context_window, max_context_window,
     effective_context_window_percent}' /tmp/codex-models.json
```

If backend support is confirmed but client metadata is stale, create a separate
catalog for **only the confirmed models**. Do not edit the automatically refreshed
`models_cache.json`. For example, if Astra has been confirmed:

```sh
mkdir -p "$HOME/.codex"
jq --arg slug gpt-6-astra '
  if any(.models[]; .slug == $slug) then
    {models: [.models[] | if .slug == $slug then
      .context_window = 1050000 | .max_context_window = 1050000
      else . end]}
  else error("Selected model is missing from the catalog") end
' /tmp/codex-models.json > "$HOME/.codex/models-1m.json"
```

Repeat with the resulting file as input for each other confirmed model used by
your profile; write to a temporary file and rename it rather than redirecting
onto the input. If a model is missing, obtain current metadata from the provider
instead of inventing an entry. Retain any lower limit when support is unconfirmed.

Create the catalog before uncommenting the block in the installed
`.codex/config.toml`, and replace the example path with the catalog's absolute
path. A missing or invalid catalog fails startup. Restart Codex after changing
the file or setting; catalog changes are applied at startup, not by per-thread
overrides. All shipped profiles remain usable without this optional file.

Inspect the loaded catalog from your trusted target project:

```sh
codex debug models | jq '.models[]
  | select(.slug == "gpt-6-astra")
  | {slug, context_window, max_context_window,
     effective_context_window_percent}'
```

This command renders catalog metadata; it does not prove the effective runtime
window or successful large requests. If the effective percentage is 95, a raw
1,050,000 window corresponds to a 997,500 client budget. The proposed 900,000
auto-compaction threshold applies to the total active context, and is below
that budget. Check the percentage for each model rather than assuming 95.
Validate actual long-context requests and compaction behavior with the provider
before relying on this configuration; that backend validation was not performed
here. If support is lower, leave the block disabled or choose supported limits.

## Key directory structure

```text
.
├── profiles/
│   ├── pro/
│   ├── plus/
│   ├── pro-max-2-subagents/
│   ├── plus-max-2-subagents/
│   ├── GPT6-SolMax-LunaMax/
│   └── GPT6-SolMedium-LunaMax/
├── guides/
├── scripts/
│   └── token_usage.py
├── tests/
│   ├── test_profiles.py
│   └── test_token_usage.py
├── AGENTS.md
├── setup.sh
├── setup.ps1
├── README.md
└── LICENSE
```

Each profile contains `codex/config.toml`, `codex/agents/*.toml`, and
`agents/skills/astra-orchestrator/SKILL.md`.

## Guides

- [Pro orchestration and manual configuration](guides/full-orchestration.md)
- [Plus profile and global setup](guides/plus-plan.md)
- [Fast iteration](guides/fast-iteration.md) and [routine coding](guides/routine-coding.md)
- [Complex repository work](guides/complex-repo-work.md)
- [Token usage and measurement](guides/token-usage.md)

## License

Licensed under the [Apache License 2.0](LICENSE).
