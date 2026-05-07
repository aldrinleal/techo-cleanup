# Techo Cleanup — Pipeline de Datos Comunitarios

Pipeline de limpieza, transformación y exposición de datos de caracterizaciones comunitarias para TECHO Colombia.

## Contexto

Somos voluntarios de TECHO que recolectan datos de campo en planillas ODS. Este proyecto transforma esas planillas en bases de datos limpias, normalizadas y anonimizadas para su uso en aplicaciones móviles y análisis interno.

### Comunidades

- Vereda El Granizal
- Manrique La Honda
- Vereda La Nueva Jerusalen

## Estructura del Proyecto

```
techo-cleanup/
├── in/                  # Archivos de entrada (datos crudos — NO en git)
│   └── sheet.ods        # Planilla principal con todas las pestañas
├── out/                 # Archivos generados por el pipeline (NO en git)
│   ├── caracterizaciones.db
│   ├── comunidades.db
│   ├── comunidad_anonimizada.db
│   └── reporte_limpieza.html
├── specs/               # PRDs, modelos de datos, arquitectura
│   ├── prompts.md       # Registro de todos los prompts usados
│   └── YYYYMMDD-*.md   # Especificaciones por fecha
├── notebooks/           # Jupyter notebooks del pipeline
│   ├── 01_ingesta.ipynb
│   ├── 02_analisis_exploratorio.ipynb
│   ├── 03_limpieza.ipynb
│   └── 04_anonimizacion.ipynb
├── src/
│   └── api/             # FastAPI backend
│       └── main.py
├── AGENTS.md            # Guía para agentes de IA
├── .env                 # Secrets locales (NO en git)
└── .env.sample          # Template de variables de entorno
```

## Setup

```bash
# 1. Clonar e instalar dependencias
git clone <repo>
cd techo-cleanup
pip install pipenv
pipenv install

# 2. Configurar variables de entorno
cp .env.sample .env
# Editar .env con los valores reales

# 3. Poner la planilla fuente en in/
cp /ruta/a/sheet.ods in/sheet.ods
```

## Ejecución del Pipeline

```bash
# Fase 1: Ingesta ODS → SQLite
pipenv run nbclick notebooks/01_ingesta.ipynb

# Fase 2: Análisis exploratorio
pipenv run jupyter notebook notebooks/02_analisis_exploratorio.ipynb

# Fase 3: Limpieza y deduplicación
pipenv run nbclick notebooks/03_limpieza.ipynb

# Fase 4: Exportación anonimizada
pipenv run nbclick notebooks/04_anonimizacion.ipynb

# Fase 5: Levantar API
pipenv run uvicorn src.api.main:app --reload
```

## API

Una vez levantada la API:

- Documentación interactiva: http://localhost:8000/docs
- `GET /comunidades` — lista con paginación
- `GET /comunidades/{id}` — detalle
- `GET /comunidades?q=<texto>` — búsqueda

## Stack Técnico

| Componente | Tecnología |
|---|---|
| Runtime | Python 3.11+, Pipenv, pyproject.toml |
| Notebooks | Jupyter, nbclick |
| LLM | LangChain + OpenRouter |
| Base de datos | SQLite3 |
| API | FastAPI + Uvicorn |
| Storage | S3 o compatible |
| Testing | pytest |

## Variables de Entorno

Ver `.env.sample` para la lista completa de variables requeridas.

## Fases del Proyecto

Ver `specs/20260413-contexto-techo-prd-limpieza-datos.md` para el PRD completo con estimados y criterios de aceptación.

| Fase | Descripción | Estimado |
|---|---|---|
| 1 | Ingesta ODS → SQLite | 3 pts |
| 2 | Análisis y modelado | 5 pts |
| 3 | Limpieza y deduplicación | 8 pts |
| 4 | Exportación anonimizada | 5 pts |
| 5 | API REST lectura | 8 pts |
| 6 | API con JSON Patches (opcional) | 13 pts |

## Contribuir

Leer `AGENTS.md` antes de trabajar en el proyecto (aplica para humanos y agentes de IA).
