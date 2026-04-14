# Registro de Prompts

Todos los prompts usados en este proyecto deben registrarse aquí antes de ejecutarse.

## Formato

```
## YYYYMMDD-HH:MM — Descripción breve
**Contexto**: Qué se intentaba lograr
**Prompt**:
> texto del prompt

**Resultado esperado**: ...
**Notas**: ...
```

---

<!-- Agregar nuevos prompts debajo de esta línea -->

## 20260413 — Configuración de CLAUDE.md como symlink a AGENTS.md

**Contexto**: Claude Code solo carga `CLAUDE.md` nativamente; `AGENTS.md` no se auto-carga. Para mantener `AGENTS.md` como fuente de verdad (compatible con otras herramientas como OpenAI Codex) y que Claude Code lo lea, se crea un symlink.

**Decisión**: `CLAUDE.md` → symlink a `AGENTS.md`

**Comando**:
```bash
ln -s AGENTS.md CLAUDE.md
```

**Notas**: Otros agentes que honran `AGENTS.md` directamente (ej. Codex) siguen funcionando. Claude Code lee `CLAUDE.md` que apunta al mismo archivo.
