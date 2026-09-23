# AGENTS.md — MDMV Harness

## Overview

`MDMV-Harness/` documenta a configuração do harness de agentes (OpenCode) da MDMV: arquivos,
conteúdo, decisões, execução e verificação. É uma área de **documentação**, não de código.

## Where To Look

| Need | Location |
|---|---|
| Configuração completa do harness | `MDMV Harness Configuration.md` |
| Pesquisa base (2026-09-22) | `../../documentos/opencode-configuracao-ideal-2026-09-22.md` |
| Revisão + GSD×Spec-kit + Fases A–G (2026-09-23) | `../../documentos/opencode-configuracao-ideal-2026-09-23.md` |
| Config global real | `~/.config/opencode/` |
| Skills curadas | `~/.agents/skills/` |

## Operational Rules

- Esta área é **fonte documental**; a fonte de verdade executável é `~/.config/opencode/` e os
  arquivos por projeto (`.opencode/`, `.specify/`, `scripts/`).
- Nunca reproduza segredos em markdown. O PAT do MCP vive em
  `~/.config/opencode/.github-mcp-token` (chmod 600).
- Ao alterar a configuração do harness, atualize `MDMV Harness Configuration.md` na mesma tarefa.

## Current Reality

- OpenCode V1 `1.18.32`; `plugin: []`; 17 skills (~1,3k tokens de metadados).
- Spec-kit por projeto; Graphify por projeto (`mdmv-linter`, `resenhafc`); impeccable em `resenhafc`.
- Subagentes `mdmv-verifier`/`mdmv-researcher`; plugins locais `state-memory` e `context-watch`.
