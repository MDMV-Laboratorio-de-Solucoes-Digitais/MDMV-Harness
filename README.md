# MDMV Harness

Configuração canônica do **harness de agentes (OpenCode)** da MDMV — Laboratório de Soluções Digitais.

## Contents

- [`MDMV Harness Configuration.md`](./MDMV%20Harness%20Configuration.md) — referência completa: arquivos, conteúdo, decisões, execução e verificação.
- [`AGENTS.md`](./AGENTS.md) — convenções desta área.

## Scope

Documentação (não código). A **fonte de verdade executável** é `~/.config/opencode/` e os
arquivos por projeto (`.opencode/`, `.specify/`, `scripts/`).

## Branches

- **`dev`** — branch principal de desenvolvimento (default).
- **`main`** — recebe apenas PRs de `dev`.

## Segurança

Nenhum segredo é versionado. O PAT do MCP GitHub vive em
`~/.config/opencode/.github-mcp-token` (chmod 600), fora deste repositório.
