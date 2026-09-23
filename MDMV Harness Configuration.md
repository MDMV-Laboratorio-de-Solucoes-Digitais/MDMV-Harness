# MDMV Harness Configuration

> **O que é:** referência canônica de **toda** a configuração do harness de agentes (OpenCode)
> da MDMV — arquivos, conteúdo, decisões, execução e verificação.
> **Data:** 2026-09-23 · **OpenCode:** `1.18.32` (V1, fork `anomalyco/opencode`)
> **Documentos relacionados:**
> - `../../documentos/opencode-configuracao-ideal-2026-09-22.md` (pesquisa base)
> - `../../documentos/opencode-configuracao-ideal-2026-09-23.md` (revisão + GSD×Spec-kit + Fases A–G)
>
> **Segurança:** nenhum segredo é reproduzido aqui. O PAT do MCP GitHub vive em
> `~/.config/opencode/.github-mcp-token` (chmod 600) e é lido via `{file:...}`.

---

## 1. Visão geral

O **MDMV Harness** é a camada de ambiente que torna os agentes determinísticos e baratos:
configuração global enxuta, skills curadas, subagentes especializados, memória de sessão,
alerta de contexto, gate de supply-chain e integrações por projeto (Spec-kit, Graphify,
impeccable). Princípio diretor: **"make the environment smarter, not the model"** —
o agente fornece a intenção; o ambiente fornece a garantia.

Alvo: **OpenCode V1 sem plugins pesados** + **Spec-kit por projeto** + **~17 skills curadas** +
**subagentes MDMV** + **`STATE.md`** + **headroom** + **supply-chain** + **Graphify/impeccable**.

---

## 2. Inventário de arquivos

| Caminho | Escopo | Papel |
|---|---|---|
| `~/.config/opencode/opencode.json` | global | Config principal (agentes, permissões, MCP, compaction) |
| `~/.config/opencode/tui.json` | global | Plugins de TUI (vazio) |
| `~/.config/opencode/AGENTS.md` | global | Convenções de orquestração/memória/Rust |
| `~/.config/opencode/agents/mdmv-verifier.md` | global | Subagente que roda o gate |
| `~/.config/opencode/agents/mdmv-researcher.md` | global | Subagente de contexto `.specify/` |
| `~/.config/opencode/plugins/state-memory.js` | global | Injeta `STATE.md` na compactação |
| `~/.config/opencode/plugins/context-watch.js` | global | Wrapper do alerta de headroom |
| `~/.config/opencode/opencode-context-watch.json` | global | Config do alerta de headroom |
| `~/.config/opencode/.github-mcp-token` | global (600) | PAT do MCP GitHub (não versionado) |
| `~/.config/opencode/package.json` | global | Deps dos plugins locais |
| `~/.agents/skills/mdmv-rust-thin/SKILL.md` | global | Skill de governança Rust |
| `~/.agents/skills/*` | global | 17 skills curadas |
| `~/.omo-archive-20260923/` | arquivo | Estado antigo do oh-my-openagent |
| `~/.agents/commands-archive-20260923/` | arquivo | Comandos Spec-kit globais antigos |
| `<projeto>/.opencode/commands/speckit.*.md` | por projeto | Spec-kit (via CLI) |
| `<projeto>/.opencode/skills/graphify/` + `plugins/graphify.js` | por projeto | Graphify |
| `<projeto>/.opencode/skills/impeccable/` + `commands/impeccable.md` | por projeto | impeccable |
| `<projeto>/.specify/memory/STATE.md` | por projeto | Memória de sessão |
| `<projeto>/scripts/supply-chain.sh` | por projeto | Gate de supply-chain |

---

## 3. Configuração global

### 3.1 `~/.config/opencode/opencode.json` (trechos relevantes)

> O bloco `provider` (inferx/thehive) permanece inalterado e é omitido aqui por brevidade.

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "default_agent": "plan",
  "instructions": ["AGENTS.md", "CONTRIBUTING.md"],
  "plugin": [],
  "lsp": true,
  "agent": {
    "build": {
      "description": "Primary coding agent for development work; go-cheap profile (OpenCode Go, DeepSeek V4.1 Flash, 1M context).",
      "mode": "primary",
      "model": "opencode-go/deepseek-v4.1-flash",
      "permission": {
        "edit": "allow", "write": "allow", "bash": "allow", "read": "allow",
        "task": { "*": "deny", "explore": "allow", "scout": "allow", "general": "allow", "mdmv-*": "allow" }
      }
    },
    "plan":      { "model": "opencode-go/deepseek-v4.1-flash" },
    "general":   { "model": "opencode-go/glm-5.3-flash" },
    "explore":   { "model": "opencode-go/mimo-v2.5" },
    "title":     { "model": "opencode-go/mimo-v2.5" },
    "summary":   { "model": "opencode-go/mimo-v2.5" },
    "compaction":{ "model": "opencode-go/deepseek-v4.1-flash" }
  },
  "model": "opencode-go/deepseek-v4.1-flash",
  "small_model": "opencode-go/mimo-v2.5",
  "autoupdate": "notify",
  "tool_output": { "max_lines": 1000, "max_bytes": 40960 },
  "compaction": { "auto": true, "tail_turns": 15, "preserve_recent_tokens": 20000 },
  "mcp": {
    "github": {
      "type": "remote",
      "url": "https://api.githubcopilot.com/mcp/",
      "enabled": true,
      "oauth": false,
      "headers": { "Authorization": "Bearer {file:/home/luis/.config/opencode/.github-mcp-token}" }
    }
  }
}
```

**Pontos-chave:** `plugin: []` (sem plugins npm); `permission.task` restringe o que o `build`
pode despachar; MCP GitHub usa **substituição por arquivo** (não `{env:}`).

### 3.2 `~/.config/opencode/tui.json`

```json
{ "plugin": [] }
```

### 3.3 `~/.config/opencode/AGENTS.md`

```markdown
# Global agent rules — MDMV

## Orchestration (keep the primary context lean)
The primary agent is an **orchestrator**, not the worker. Delegate heavy or wide work to subagents:

- `@explore` — fast read-only search inside the repo.
- `@scout` — read-only research on external docs/dependencies (clones into cache).
- `@general` — multi-step tasks that may change files.
- `@mdmv-verifier` — run the deterministic gate (`./verify.sh` / cargo) and report PASS/FAIL. Do **not** self-certify completion; verify.
- `@mdmv-researcher` — ground a question in `.specify/` + repo before planning.

Avoid reading large files or running broad searches in the primary session when a subagent can do it.

## Memory (session continuity)
- When a project has `.specify/memory/STATE.md`, **read it at the start of any non-trivial task** and **update it when you finish** (position, decisions, blockers, next step). Keep it ≤1 page.
- Do not preload `STATE.md` into context; load it on demand. A plugin preserves it across compaction automatically.

## Rust / MDMV
- Follow the `mdmv-rust-thin` skill for Rust work.
- Before marking a task complete, run the verification gate (via `@mdmv-verifier` when non-trivial).

## Tooling
- No third-party plugins in the critical path; prefer native OpenCode features.
```

### 3.4 Subagentes

`~/.config/opencode/agents/mdmv-verifier.md`:

```markdown
---
description: "Run the deterministic MDMV verification gate in an isolated context and return a pass/fail digest with diagnostics. Use after implementing or refactoring Rust code, and before declaring a task complete."
mode: subagent
model: opencode-go/deepseek-v4.1-flash
temperature: 0.0
permission:
  edit: deny
  read: allow
  glob: allow
  grep: allow
  list: allow
  bash:
    "*": deny
    "./verify.sh": allow
    "./verify.sh *": allow
    "cargo nextest*": allow
    "cargo clippy*": allow
    "cargo fmt*": allow
    "cargo check*": allow
    "cargo test*": allow
    "git status*": allow
    "git diff*": allow
---

You are the MDMV verifier. You never change code; you prove whether it passes.

## Mission
Run the project's canonical gate and report the truth. The environment validates the code, not opinion.

## Procedure
1. If `./verify.sh` exists at the repo root, run it. This is the canonical gate (same script as CI and the pre-commit hook).
2. Otherwise run, in order: `cargo fmt --all -- --check`, then `cargo clippy --all-targets -- -D warnings`, then `cargo nextest run` (fallback `cargo test`).
3. Do not edit, patch, or "fix" anything. If something fails, capture the exact error.

## Output (strict)
- VERDICT: PASS | FAIL
- GATE: which commands ran
- FAILURES: for each, `file:line` + the exact diagnostic + the offending lint/error code
- SUGGESTED NEXT STEP: one line (the fix belongs to the caller)

Do not speculate beyond the observed output.
```

`~/.config/opencode/agents/mdmv-researcher.md`:

```markdown
---
description: "Answer MDMV project questions by reading the repo, .specify/ artifacts (constitution, specs, plans, tasks) and AGENTS.md, returning a short cited digest. Use before planning a feature or when project conventions/specs must be grounded."
mode: subagent
model: opencode-go/glm-5.3-flash
temperature: 0.1
permission:
  edit: deny
  bash: deny
  read: allow
  glob: allow
  grep: allow
  list: allow
  webfetch: allow
  websearch: allow
---

You are the MDMV researcher. You gather grounded context and return a digest; you never modify files.

## Sources, in priority order
1. Project `AGENTS.md` and `.specify/memory/constitution.md` (binding rules).
2. `.specify/specs/**` (`spec.md`, `plan.md`, `tasks.md`, `research/`) for the feature at hand.
3. The repo itself (read/grep/glob); cite `path:line`.
4. External docs (webfetch/websearch) only when the repo is silent.

## Rules
- Prefer the repo over training memory: a claim from memory is `[ASSUMED]`, from a file is `[VERIFIED: path:line]`.
- Do not paste large files; summarize and cite.
- Never edit. Return findings to the caller.

## Output
- QUESTION
- FINDINGS (bulleted, each with a citation)
- CONFLICTS / UNKNOWNS (explicit)
- RECOMMENDED NEXT STEP (one line)
```

### 3.5 Plugins locais

`~/.config/opencode/plugins/state-memory.js`:

```javascript
import { existsSync, readFileSync } from "node:fs"
import { join } from "node:path"

const CANDIDATES = ["STATE.md", "state.md"]

function findStateFile(root) {
  for (const name of CANDIDATES) {
    const path = join(root, ".specify", "memory", name)
    if (existsSync(path)) return path
  }
  return null
}

export const StateMemoryPlugin = async ({ directory, worktree }) => {
  return {
    "experimental.session.compacting": async (_input, output) => {
      const roots = [worktree, directory].filter(Boolean)
      let path = null
      for (const root of roots) {
        path = findStateFile(root)
        if (path) break
      }
      if (!path) return
      const content = readFileSync(path, "utf8").trim()
      if (!content) return
      output.context.push(
        "## Project state (from .specify/memory/STATE.md — preserve verbatim across compaction)\n\n" +
          content,
      )
    },
  }
}
```

`~/.config/opencode/plugins/context-watch.js`:

```javascript
// OpenCode V1 (anomalyco 1.18.x) requires the named-export plugin API.
// The npm package's `default` export breaks this build's loader, so we re-export
// its named plugin function through a thin, auditable wrapper.
// Package: opencode-context-watch (MIT) — config: ~/.config/opencode/opencode-context-watch.json
import { ContextWatchPlugin } from "opencode-context-watch"

export const ContextWatch = ContextWatchPlugin
```

`~/.config/opencode/opencode-context-watch.json`:

```json
{
  "warnPercent": 0.75,
  "warnTokens": 900000,
  "rearmPercent": 5,
  "toast": true,
  "verbose": false,
  "postCompactContinue": true,
  "postCompactMsg": "[context-watch] A sessão foi compactada. Continue de onde parou, mantendo as respostas concisas."
}
```

`~/.config/opencode/package.json`:

```json
{
  "dependencies": {
    "@opencode-ai/plugin": "1.18.32",
    "opencode-context-watch": "^0.1.3"
  }
}
```

### 3.6 Segredo do MCP

- Arquivo: `~/.config/opencode/.github-mcp-token` (`chmod 600`, sem newline).
- Consumido por `{file:/home/luis/.config/opencode/.github-mcp-token}` no header do MCP.
- **Não** usar `{env:GITHUB_MCP_TOKEN}`: neste ambiente o processo do OpenCode não herda o
  ambiente do shell interativo (resolve vazio → HTTP 400).

---

## 4. Configuração por projeto

### 4.1 Spec-kit (via CLI, não universal)

```bash
specify init <projeto> --integration opencode   # cria .opencode/commands/speckit.*.md
```
- Instalado por projeto em `.opencode/commands/` (10 comandos: constitution, specify, clarify,
  plan, tasks, analyze, implement, converge, checklist, taskstoissues).
- Os comandos globais antigos (`~/.agents/commands/speckit.*`) foram **arquivados**.
- Não subir para OpenCode 2.x (bug Spec-kit #4639).

### 4.2 Graphify (por projeto)

```bash
graphify install --project --platform opencode
```
Produz: `.opencode/skills/graphify/` (SKILL.md + references), `.opencode/plugins/graphify.js`
(hook `tool.execute.before` que injeta um lembrete na 1ª chamada bash quando
`graphify-out/graph.json` existe), `.opencode/opencode.json` (registra o plugin) e uma seção
`## graphify` no `AGENTS.md` do projeto.

Instalado em: `mdmv-linter`, `resenhafc`. `graphify-out/` está no `.gitignore`.

### 4.3 impeccable (por projeto frontend)

```bash
npx impeccable install --providers=opencode --scope=project --project --yes
```
Produz: `.opencode/skills/impeccable/` (SKILL.md + reference/ + scripts/), `.opencode/commands/impeccable.md`.
O binário de ~16MB (`scripts/bin/`) e `scripts/data/` estão gitignorados em `.opencode/.gitignore`.
Instalado em: `resenhafc`. Retomar contexto com `/impeccable init` na próxima sessão.

### 4.4 `STATE.md` (memória)

Local: `<projeto>/.specify/memory/STATE.md`. Lido sob demanda (AGENTS.md) e injetado na
compactação pelo plugin `state-memory.js`. Template em §9.

### 4.5 Gate de supply-chain

Local: `<projeto>/scripts/supply-chain.sh` (ver §6).

---

## 5. Skills

Pool global: **17 skills** em `~/.agents/skills/`, **~5.125 bytes** de metadados residentes
(≈1,3k tokens).

```
codebase-design  coding-guidelines  domain-modeling  intended-vs-implemented
mdmv-rust-thin  receiving-code-review  requesting-code-review  research
shipping-artifacts  systematic-debugging  tailwind-design-tokens
test-driven-development  to-tickets  ui-styling  unsafe-checker
verification-before-completion  writing-plans
```

`~/.agents/skills/mdmv-rust-thin/SKILL.md`:

```markdown
---
name: mdmv-rust-thin
description: "Deterministic Rust code governance for MDMV. Use in Rust implementation and refactoring tasks. Keywords: rust, clippy, pedantic, lint, unsafe, unwrap, expect, thiserror, anyhow, nextest, verify, cargo, governance, MDMV"
---

# MDMV Rust Governance

## Golden Rule
The environment validates the code, not your opinion. Compilation under the pinned toolchain is the absolute truth.

## Non-Negotiable Constraints
1. `unsafe_code = "forbid"` across all production code.
2. `clippy::pedantic = "deny"` — treat all warnings as fatal errors.
3. `.unwrap()` and `.expect()` are forbidden. Handle errors via `Result<T, E>`, the `?` operator, and the `thiserror`/`anyhow` crates.
4. NEVER modify Cargo.toml or clippy.toml to disable lints. If a proc-macro requires an isolated exception, use strictly:
   `#[expect(clippy::lint_name, reason = "Auditable technical rationale")]`

## Verification Cycle
Before marking any task in tasks.md as complete:
1. Run `./verify.sh` (the same script used in CI).
2. If there is a lint or compilation failure, resolve the error without adding external dependencies.
3. Run `cargo nextest run` to ensure zero regressions.

## Dependencies (supply-chain)
Before adding or upgrading any dependency:
1. Run the canonical gate: `./scripts/supply-chain.sh` (or, if absent, `cargo deny check && cargo audit`).
2. Pin the version explicitly; never leave a floating `*` or unbounded range.
3. NEVER auto-substitute a similarly-named package (anti-slopsquatting). If a package fails the gate, STOP and surface the finding — do not swap in an alternative without explicit human approval.
4. Record the rationale for a new dependency in the commit message.
```

---

## 6. Gate de supply-chain

`mdmv-linter/scripts/supply-chain.sh` (canônico; deve ser chamado por `verify.sh`/pre-commit/CI):

```bash
#!/usr/bin/env bash
# ============================================
# MDMV SUPPLY-CHAIN GATE
# ============================================
# Canonical, deterministic gate for third-party dependency risk.
# Designed to be called by verify.sh / pre-commit / CI (agent→git→CI parity).
#
# Usage:
#   ./supply-chain.sh            # local: missing tools are warnings
#   ./supply-chain.sh --strict   # CI: missing tools are failures
#
# Checks (in order):
#   1. cargo deny check   — policy: advisories, bans, licenses, sources
#   2. cargo audit        — live RustSec advisories against the lockfile
#   3. osv-scanner        — OSV.dev across Cargo.lock + package-lock/pnpm-lock/yarn
#
# Exit: 0 = clean, 1 = findings or (strict) missing tool.

set -uo pipefail

STRICT=0
[ "${1:-}" = "--strict" ] && STRICT=1

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'
info()  { echo -e "${GREEN}[supply-chain]${NC} $1"; }
warn()  { echo -e "${YELLOW}[supply-chain]${NC} $1"; }
fail()  { echo -e "${RED}[supply-chain]${NC} $1"; }

FAILED=0

missing() {
  if [ "$STRICT" -eq 1 ]; then
    fail "missing tool: $1 ($2)"
    FAILED=1
  else
    warn "skipping: $1 not installed ($2)"
  fi
}

# 1. cargo deny — policy engine (advisories/bans/licenses/sources)
if [ -f "Cargo.lock" ]; then
  if command -v cargo-deny >/dev/null 2>&1; then
    info "cargo deny check"
    cargo deny check || { fail "cargo deny found issues"; FAILED=1; }
  else
    missing cargo-deny "cargo install cargo-deny"
  fi

  # 2. cargo audit — live RustSec advisories
  if command -v cargo-audit >/dev/null 2>&1; then
    info "cargo audit"
    cargo audit || { fail "cargo audit found advisories"; FAILED=1; }
  else
    missing cargo-audit "cargo install cargo-audit"
  fi
else
  warn "no Cargo.lock; skipping cargo deny/audit"
fi

# 3. osv-scanner — multi-ecosystem (crates.io + npm/pnpm/yarn)
if command -v osv-scanner >/dev/null 2>&1; then
  info "osv-scanner scan source -r ."
  osv-scanner scan source -r . --allow-no-lockfiles || { fail "osv-scanner found vulnerabilities"; FAILED=1; }
else
  missing osv-scanner "mise use -g aqua:google/osv-scanner (or: go install github.com/google/osv-scanner/v2/cmd/osv-scanner@latest)"
fi

# 4. Optional: cargo-vet (trust/audit records) — informational only
if command -v cargo-vet >/dev/null 2>&1 && [ -f "supply-chain/config.toml" ]; then
  info "cargo vet (trust records present)"
  cargo vet || { fail "cargo vet found unaudited dependencies"; FAILED=1; }
fi

if [ "$FAILED" -eq 0 ]; then
  info "OK — no supply-chain findings"
  exit 0
fi
fail "supply-chain gate FAILED"
exit 1
```

---

## 7. Registro de alterações

### 7.1 Decisões (D1–D9)

| # | Decisão | Implementação |
|---|---|---|
| D1 | Spec-kit **por projeto via CLI** (não universal) | `.opencode/commands/speckit.*` por repo; globais arquivados |
| D2 | PAT **fora do JSON** | `{file:~/.config/opencode/.github-mcp-token}` |
| D3 | Remover `oh-my-openagent` | `tui.json` → `{"plugin": []}` |
| D4 | Arquivar `~/.omo/` + symlinks `.codegraph` | `~/.omo-archive-20260923/`; 3 symlinks removidos |
| D5 | Limpar `mdmv-rust-thin` | `user-invocable:false` removido |
| D6 | Graphify **por projeto** | `graphify install --project --platform opencode` |
| D7 | impeccable como **única** skill de design | por projeto frontend (`resenhafc`) |
| D8 | **Não** instalar pacotes de domínio | pm-skills/marketingskills/web-quality |
| D9 | Spec-kit canônico; **GSD descartado** | emprestar 4 ideias (ver doc 2026-09-23) |

### 7.2 Execução (Fases A–F)

| Fase | Mudanças |
|---|---|
| **A — Coerência** | `tui.json` limpo; `~/.omo/` arquivado; symlinks `.codegraph` removidos; `mdmv-rust-thin` limpo; PAT → arquivo; `speckit.*` globais arquivados |
| **B — Subagentes** | `mdmv-verifier.md`, `mdmv-researcher.md`, `build.permission.task`, `~/.config/opencode/AGENTS.md` |
| **C — Memória** | `plugins/state-memory.js`, `.specify/memory/STATE.md` (mdmv-linter), seção Memory no AGENTS.md global |
| **D — Headroom** | `opencode-context-watch` (npm) + `plugins/context-watch.js` (wrapper V1) + `opencode-context-watch.json`; `opencode.json` plugin de volta a `[]` |
| **E — Supply-chain** | `osv-scanner` 2.6.0 (mise); `scripts/supply-chain.sh`; protocolo de dependências na skill; MCP corrigido para `{file:...}` |
| **F — Graphify + impeccable** | Graphify por projeto em `mdmv-linter` e `resenhafc`; impeccable em `resenhafc`; gitignore de `graphify-out/` e do binário do impeccable |

### 7.3 Arquivamentos

| De | Para |
|---|---|
| `~/.omo/` | `~/.omo-archive-20260923/` |
| `~/.agents/commands/speckit.*.md` | `~/.agents/commands-archive-20260923/` |
| `~/.config/opencode/.codegraph` | removido (apontava para `~/.omo`) |
| `MDMV/.codegraph`, `mdmv-linter/.codegraph` | removidos |

Backups da Fase A: `~/.config/opencode/backups/phase-a-20260923/`.

---

## 8. Verificação (evidências)

| Experimento | Resultado |
|---|---|
| MCP GitHub com `{file:...}` | `opencode mcp list` → `✓ github connected` |
| Subagente `mdmv-verifier` | rodou `./verify.sh` → PASS; `ls` **negado** por `permission` |
| `state-memory.js` | unit test: `CASE1 injected: true` / `CASE2 no-op: true`; diretório de plugins varrido (marker) |
| `context-watch` | hooks registrados; unit test injeta com limiar baixo, no-op com alto |
| `supply-chain.sh` | repo sem lockfile → EXIT 0; `resenhafc` → EXIT 1 (vulns reais) |
| Graphify plugin | `pwd` precedido de `[graphify] knowledge graph at graphify-out/…` |
| API de plugins | named export V1 carrega; `default` quebra o loader (teste A/B) |

---

## 9. Template de `STATE.md`

```markdown
# STATE — <projeto>

> Memória de sessão (≤1 página). Leia no início de tarefa não-trivial; atualize ao encerrar.
> Atualizado: AAAA-MM-DD

## Posição
- Fase/milestone atual; o que está pronto vs pendente.

## Decisões ativas
- Decisões que a próxima sessão precisa respeitar.

## Bloqueios / riscos
- O que impede o progresso; dependências externas.

## Próximo passo
- A próxima ação concreta.
```

---

## 10. Reprodução (do zero)

```bash
# 1. Config global
#    - editar ~/.config/opencode/opencode.json (plugin: [], agentes, MCP {file:...})
#    - tui.json = {"plugin": []}
#    - criar ~/.config/opencode/AGENTS.md
#    - criar agents/mdmv-verifier.md e agents/mdmv-researcher.md
#    - criar plugins/state-memory.js e plugins/context-watch.js
#    - criar .github-mcp-token (chmod 600)

# 2. Headroom
cd ~/.config/opencode && bun add opencode-context-watch
#    criar opencode-context-watch.json

# 3. Supply-chain
mise use -g aqua:google/osv-scanner
cargo install cargo-deny cargo-audit cargo-vet   # se ausentes

# 4. Por projeto (ex.: resenhafc)
specify init . --integration opencode            # Spec-kit
graphify install --project --platform opencode   # Graphify
npx impeccable install --providers=opencode --scope=project --project --yes  # impeccable
#    criar .specify/memory/STATE.md
#    copiar scripts/supply-chain.sh
#    gitignore: graphify-out/ ; .opencode/.gitignore: skills/impeccable/scripts/bin/ e data/
```

---

## 11. Pendências

- [ ] WikiSkill/SkillForge no CI (timebox ≤2 dias): traces → `.wiki/raw/` → job propõe skill → gate.
- [ ] Ligar `supply-chain.sh` ao `verify.sh`/CI na F1 do `mdmv-linter`.
- [ ] Enriquecer o `AGENTS.md` do `mdmv-linter` (hoje só a seção graphify).
- [ ] Construir o grafo do `resenhafc` (`graphify .`) e atualizar deps vulneráveis.
- [ ] Trigger test (R5) das skills; podar se <80%.
- [ ] Experimento R1 (`permission.skill deny`).
- [ ] Limpar cache residual (`~/.cache/opencode/packages/oh-my-openagent@latest`, `superpowers@…`).
- [ ] `/impeccable init` no `resenhafc` na próxima sessão.

---

## 12. Gotchas conhecidos

1. **API de plugins = V1 (named export)** neste build. `default export` que não seja
   `{ server() }` não carrega → use wrapper para pacotes npm.
2. **`{env:VAR}` não propaga** ao OpenCode aqui → use `{file:...}` para segredos.
3. **`osv-scanner` sai 128** em "no package sources" → `--allow-no-lockfiles`.
4. **Graphify** só age se `graphify-out/graph.json` existir (rodar `graphify .` 1×).
5. **impeccable** traz binário de ~16MB → gitignorar `scripts/bin/` e `scripts/data/`.
6. **Config não é hot-reload** → reiniciar o OpenCode após editar.

---

## 13. Arquitetura do pipeline e cherry-picks (CenterOS)

> Origem: análise do vídeo *"CenterOS is the Best Free AI Harness Out There"* (John Elder).
> O CenterOS é um **scaffold Markdown cloneável** (não um runtime de agente). Ele **valida a
> tese 90-10** ("code = determinístico; AI = juízo") e rendeu 3 cherry-picks. Não é adotado
> inteiro (seria um segundo dono de estado e bootstrap always-on).

### 13.1 Camadas (modelo corrigido)

O **orquestrador é o `fabro`**; a **harness é transversal**; e o **`crsdd-fabro` é o repo que
implementa as três rooms** (Contaminated agora; **Vault e Clean Room depois, no mesmo repo**),
como **estágios sequenciais** — não nós irmãos em repos separados.

| Camada | O que é | Onde |
|---|---|---|
| **Orquestrador** | `fabro` (`fabro-sh/fabro`) — runs duráveis, grafos `.fabro`, checkpoints, aprovação (`fabro run/events/logs/approve/steer`) | binário `fabro` |
| **Harness (transversal)** | ambiente que envolve tudo: OpenCode (agents + subagents + config + skills) + Spec-kit + mdmv-linter + `STATE.md` | `~/.config/opencode/`, `.opencode/`, `.specify/` |
| **Pipeline (rooms)** | `crsdd-fabro`: **Contaminated Room → Vault → Clean Room** (estágios sequenciais) | repo `crsdd-fabro` |
| **Motor de execução** | `casv-rust` — preenche `todo!()`/`unimplemented!()` (gap filler, 4 gates próprios) | repo `casv-rust` |

- `crsdd-fabro` **hospeda as três rooms**; hoje **só a Contaminated está implementada**
  (o `validator-fabro` é o nó de comando *dessa* room). Vault e Clean Room são planejadas.
- `mdmv-linter` é **um componente de governança** da harness (lint/CI/presets/fixture) — **não**
  é a harness inteira. O repo **`MDMV-Harness`** é a documentação do ambiente.
- Spec-kit é camada própria (workflow de spec), separada do mdmv-linter.

```
ORQUESTRADOR: fabro ── grafos .fabro, runs duráveis, checkpoints, aprovação
   │
   ├─ HARNESS TRANSVERSAL: OpenCode (agents+subagents+config+skills) · Spec-kit · mdmv-linter · STATE.md
   │
   └─ PIPELINE em crsdd-fabro (rooms sequenciais):
        Contaminated Room  [IMPLEMENTADO · LLM-free] ── contrato versionado ──▶
        Vault              [planejado] ── Spec-kit: constitution→specify→clarify→plan→checklist→tasks→analyze
        Clean Room         [planejado] ── TDD estrito:
             1. escrever TESTES da vertical slice      (agente)
             2. speckit.implement → esqueleto + todo!() (agente)
             3. casv-rust preenche os todo!()           (repo casv-rust · 4 gates)
             4. rodar TESTES de aceitação               (gate do Fabro)
             5. speckit.converge ↔ speckit.implement    (loop até convergir)
```

**TDD × casv-rust:** o `speckit.tasks` (com testes-antes-de-implementar e vertical slices) é a
política da Clean Room; o `casv-rust` entra **entre** o esqueleto (`todo!()`) e a execução dos
testes. Testes de aceitação ficam no gate do Fabro; proptest/unit são os gates internos do
casv-rust — complementares.

### 13.2 Cherry-picks e encaixe

| Cherry-pick | Projeto(s) | Etapa | Estado |
|---|---|---|---|
| **1. `LOG.md` append-only** | **crsdd-fabro** (piloto) + **mdmv-linter** (convenção) | Nó determinístico + transversal | **Implementado** |
| **2. `CONTEXT.md` por diretório** | casv-rust, crsdd-fabro, macro-repo | Nós + harness | Planejado |
| **3. Framing 90-10** | mdmv-linter, casv-rust, crsdd-fabro | Documentação | Registrado |

### 13.3 Cherry-pick 1 (implementado)

**`mdmv-linter` — dona da convenção:**
- `docs/log-convention.md` — especificação (formato, statuses, append-only).
- `assets/LOG.template.md` — template.
- `scripts/check-logs.sh` — validador (heading, status fechado, campos obrigatórios,
  monotonicidade; ignora comentários HTML).
- `scripts/log.sh` — helper de append (timestamp UTC determinístico).

**`crsdd-fabro` — piloto:**
- Vendoriza os 4 artefatos; cria `LOG.md` (root) + `crates/validator-{core,cli,fabro}/LOG.md`.
- **Gate 16** em `scripts/ci.sh`: `16/16 logs — LOG.md append-only convention`
  (passa vacuamente enquanto não há `LOG.md`).
- Documentado no `AGENTS.md` (seção *LOG.md (append-only component logs)*).

**Formato:** `## <ISO-8601-UTC> [<status>]` + `- run:` / `- actor:` / `- summary:`;
status ∈ `ran-start` | `ran-complete` | `ran-failed`. Complementa (não substitui) o event log
do Fabro (`fabro events`/`fabro logs`) e o `.wiki/raw/` do harness.

### 13.4 Onde **não** encaixa

- Lógica de **orquestração dentro do `crsdd-fabro`** — as rooms são estágios; quem orquestra
  (runs, retomada, aprovação) é o **Fabro**.
- Spec-kit **dentro do `mdmv-linter`** (camadas separadas).
- `BOOTSTRAP.md` always-on (não adotar; manter `STATE.md` lazy).
