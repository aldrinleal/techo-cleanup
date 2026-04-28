# Registro de Prompts

Dos tipos de entradas:
1. **Prompts de sesión Claude Code** — las instrucciones que el usuario le dio a Claude durante el desarrollo del proyecto (trazabilidad del proceso de construcción).
2. **Prompts LLM del pipeline** — prompts enviados a modelos como parte de los notebooks/scripts del producto.

## Formato — Sesión Claude Code

```
## YYYYMMDD-HH:MM — Descripción breve
> texto del prompt
```

## Formato — Prompt LLM del pipeline

```
## YYYYMMDD-HH:MM — Descripción breve
**Contexto**: Qué se intentaba lograr
**Prompt**:
> texto del prompt
**Resultado esperado**: ...
**Notas**: ...
```

## One-liner para extraer prompts de sesión

```bash
jq -rs "[.[] | select(.project == \"$PWD\") | {ts: (.timestamp/1000|strftime(\"%Y-%m-%d %H:%M\")), prompt: .display}] | .[]" ~/.claude/history.jsonl
```

---

<!-- Agregar nuevos prompts debajo de esta línea -->

## 2026-04-27 — Spec de deduplicación con LLM opcional
> analise - y crea - una spec para lidiar com la fase 2, la de duplicados. considera el uso opcionalmente de un LLM para ayudar a pillarlos

## 2026-04-14 — Cambio de concepto de anonimización (token hash)
> ahora cambia el concepto de anonimizacion: no hay que excluir el campo. sino implementar este algoritmo:
> hacer un split en tokens. para cada token, remplazar por pseudocodigo anon_token(s): s.lower().sinacentos + sha1[:8]
> planear, evaluar y hacer una spec antes de implementar.

---

## Sesiones Claude Code — 2026-04-14

## 2026-04-14 03:03 — Inicialización del proyecto
> empiece creando un AGENTS.md basico con la instruction de siempre modificar spec/prompts.md para cada nuevo prompt. inicialize un repo git con gitignore en `in/` y `out/`... [prompt completo de inicialización del proyecto con stack, PRD, comunidades y estructura]

## 2026-04-14 03:32 — Compatibilidad AGENTS.md / CLAUDE.md
> do you honor `AGENTS.md` file or a symlink to it from `CLAUDE.md` is needed?

## 2026-04-14 03:33 — Symlink CLAUDE.md → AGENTS.md
> do A. also update `specs/prompts.md` accordingly

## 2026-04-14 03:40 — Implementación US-01
> implementa US-01 porfa

## 2026-04-14 03:58 — Actualizar specs con contexto del screenshot de WhatsApp
> actually actualiza los specs con eso / could you read the screenshot again and update the specs?

## 2026-04-14 04:15 — Validación pre-commit del notebook
> prior to commit nbclick run it

## 2026-04-14 04:19 — Registro de prompts de sesión
> update `specs/prompts.md` to my previous claude sessions (could you create a simple one-liner to jq...)

---

## 20260413 — Configuración de CLAUDE.md como symlink a AGENTS.md

**Contexto**: Claude Code solo carga `CLAUDE.md` nativamente; `AGENTS.md` no se auto-carga. Para mantener `AGENTS.md` como fuente de verdad (compatible con otras herramientas como OpenAI Codex) y que Claude Code lo lea, se crea un symlink.

**Decisión**: `CLAUDE.md` → symlink a `AGENTS.md`

**Comando**:
```bash
ln -s AGENTS.md CLAUDE.md
```

**Notas**: Otros agentes que honran `AGENTS.md` directamente (ej. Codex) siguen funcionando. Claude Code lee `CLAUDE.md` que apunta al mismo archivo.
