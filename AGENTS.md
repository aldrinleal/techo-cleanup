# AGENTS.md — Guía para Agentes de IA

## Regla Principal

**Siempre que se agregue, modifique o refine un prompt, actualizar `specs/prompts.md` con el nuevo prompt antes de ejecutarlo.**

Esto garantiza trazabilidad de todos los prompts usados en el proyecto.

## Estructura del Proyecto

```
techo-cleanup/
├── in/          # Archivos de entrada (ignorado por git, excepto .gitkeep)
├── out/         # Archivos de salida generados (ignorado por git, excepto .gitkeep)
├── specs/       # Especificaciones y PRDs — formato YYYYMMDD-descripcion.md
├── notebooks/   # Jupyter notebooks de procesamiento
├── src/         # Código fuente Python
├── AGENTS.md    # Este archivo
├── README.md    # Documentación del proyecto
├── .env         # Variables de entorno (NO commitear — ver .env.sample)
└── .env.sample  # Versión anonimizada de .env para compartir
```

## Reglas de Trabajo

1. **Prompts**: Registrar en `specs/prompts.md` antes de ejecutar.
2. **Specs**: Nuevas especificaciones en `specs/YYYYMMDD-descripcion.md`.
3. **Secrets**: Nunca commitear `.env`. Mantener `.env.sample` actualizado con claves vacías o placeholders.
4. **in/ y out/**: No commitear archivos en estas carpetas (contienen datos reales de beneficiarios).
5. **Anonimización**: Los datos personales nunca deben aparecer en código, logs ni commits.

## Stack Técnico

- **Runtime**: Python con Pipenv / pyproject.toml
- **Notebooks**: Jupyter + nbclick para ejecución CLI de notebooks
- **LLM**: LangChain conectado vía OpenRouter
- **DB**: SQLite3 (local), FastAPI (API backend)
- **Storage**: S3 o similar para artefactos
- **Datos fuente**: Planillas `.ods` en `in/`

## Variables de Entorno Requeridas

Ver `.env.sample` para la lista completa.
