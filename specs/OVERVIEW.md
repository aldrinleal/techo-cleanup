# OVERVIEW — Techo Cleanup: Pipeline de Datos Comunitarios

## Descripción Breve

**Techo Cleanup** es un pipeline de datos que transforma planillas ODS con entrevistas de campo (caracterizaciones de familias en asentamientos informales) en bases de datos relacionales limpias, normalizadas y auditables.

Los datos provienen de **dos Google Sheets** distintos:
1. **Google Sheet 1** — alimentado por un Google Form de encuesta inicial a familias → pestaña "Caracterizaciones"
2. **Google Sheet 2** — entrevistas de seguimiento por comunidad, ingreso manual → 3 pestañas de comunidad

Ambos se exportan en un único ODS que sirve de entrada al pipeline.

Produce dos artefactos principales: una base de datos interna completa para coordinadores, y una versión anonimizada sin datos personales (PII) expuesta mediante una API REST para consumo desde aplicaciones móviles de campo.

### Flujo de Origen

```
Google Form ──► Google Sheet 1 (Caracterizaciones) ──────────────────╮
                                                                       ► in/sheet.ods ──► Pipeline
Seguimiento manual ──► Google Sheet 2 (3 comunidades) ───────────────╯
```

## Funciones Principales

| Función | Descripción |
|---|---|
| **Ingesta** | Lee planillas `.ods` con datos de múltiples comunidades y las carga en SQLite sin pérdida de datos |
| **Limpieza** | Normaliza texto, fechas y codificación; detecta y marca duplicados con fuzzy matching por cédula y nombre |
| **Anonimización** | Elimina o hashea campos PII (nombres, cédulas, teléfonos, direcciones exactas) y reemplaza IDs por UUIDs no correlacionables |
| **API REST** | Sirve los datos anonimizados con paginación y búsqueda; acepta actualizaciones sobre datos internos almacenando cada cambio como JSON Patch (RFC 6902) |
| **Trazabilidad** | Toda modificación a un registro genera un patch almacenado con timestamp y autor, reversible de forma individual |

---

## Casos de Uso Principales

### UC-1 — Coordinador regenera la base de datos desde el ODS

El equipo de datos recibe una nueva versión de `sheet.ods` con datos frescos de campo. El coordinador ejecuta el pipeline completo para actualizar las bases SQLite, revisar el reporte de limpieza y publicar la nueva versión anonimizada.

```mermaid
sequenceDiagram
    actor Coord as Coordinador
    participant NB as Pipeline (nbclick)
    participant ODS as in/sheet.ods
    participant DBi as out/caracterizaciones_limpio.db
    participant DBa as out/comunidad_anonimizada.db
    participant Rep as out/reporte_limpieza.html

    Coord->>ODS: Coloca nueva versión del ODS
    Coord->>NB: nbclick run 01_ingesta
    NB->>ODS: Lee pestañas (caracterizaciones + 3 comunidades)
    NB-->>DBi: Escribe datos crudos en SQLite

    Coord->>NB: nbclick run 03_limpieza
    NB->>DBi: Lee datos crudos
    NB-->>DBi: Escribe datos normalizados + flags de duplicados
    NB-->>Rep: Genera reporte HTML con conteos

    Coord->>NB: nbclick run 04_anonimizacion
    NB->>DBi: Lee datos limpios
    NB-->>DBa: Escribe versión sin PII con UUIDs

    Coord->>Rep: Revisa reporte de calidad
```

---

### UC-2 — App móvil consulta datos de comunidad

Un voluntario en campo abre la app móvil. La app consulta la API REST para obtener el listado de familias de su comunidad asignada y busca un registro por texto.

```mermaid
sequenceDiagram
    actor Vol as Voluntario (App Móvil)
    participant App as React Native App
    participant API as FastAPI Backend
    participant DBa as comunidad_anonimizada.db

    Vol->>App: Abre sección "Mi Comunidad"
    App->>API: GET /comunidades?comunidad=granizal&limit=50
    API->>DBa: SELECT ... WHERE comunidad='granizal' LIMIT 50
    DBa-->>API: Rows (sin PII)
    API-->>App: JSON con lista paginada
    App-->>Vol: Muestra listado de familias

    Vol->>App: Busca "calle principal"
    App->>API: GET /comunidades?q=calle+principal
    API->>DBa: FTS / LIKE query
    DBa-->>API: Resultados
    API-->>App: JSON
    App-->>Vol: Muestra resultados filtrados
```

---

### UC-3 — Auditor revisa historial de cambios de un registro

Un coordinador actualiza datos de una familia vía API (v2). Más tarde, el auditor revisa qué cambió, cuándo y quién lo modificó; decide revertir un patch específico.

```mermaid
sequenceDiagram
    actor Coord as Coordinador
    actor Aud as Auditor
    participant API as FastAPI Backend
    participant DBi as caracterizaciones_limpio.db

    Coord->>API: PATCH /caracterizaciones/42 {num_personas: 5}
    API->>DBi: Lee registro actual
    API->>DBi: Aplica cambio + guarda JSON Patch en tabla patches
    API-->>Coord: 200 OK

    Aud->>API: GET /caracterizaciones/42/historial
    API->>DBi: SELECT * FROM patches WHERE entidad_id=42
    DBi-->>API: Lista de patches con timestamps y autores
    API-->>Aud: JSON con historial completo

    Aud->>API: DELETE /caracterizaciones/42/historial/patch_7
    API->>DBi: Revierte patch_7, elimina registro
    API-->>Aud: 200 OK — registro restaurado
```

---

## Modelo de Datos

### Diagrama Entidad-Relación

```mermaid
erDiagram
    COMUNIDAD {
        integer id PK
        text nombre
        text region
        text descripcion
    }

    FAMILIA {
        integer id PK
        integer comunidad_id FK
        text nombre_jefe_hogar
        text cedula
        text telefono
        text direccion
        integer num_personas
        text estrato
        date fecha_caracterizacion
        text encuestador
        boolean es_duplicado
        integer duplicado_de FK
        timestamp created_at
        timestamp updated_at
    }

    CARACTERIZACION {
        integer id PK
        integer familia_id FK
        text fuente
        text material_paredes
        text material_piso
        text material_techo
        boolean acceso_agua
        boolean acceso_energia
        boolean acceso_alcantarillado
        text tipo_tenencia
        integer num_cuartos
        boolean hacinamiento
        float score_prioridad
        text observaciones
        timestamp registrado_en
    }

    ENTREVISTA_SEGUIMIENTO {
        integer id PK
        integer familia_id FK
        integer comunidad_id FK
        date fecha_visita
        text encuestador
        text observaciones_campo
        text situacion_actual
        boolean requiere_intervencion
        timestamp registrado_en
    }

    PATCH_LOG {
        integer id PK
        text entidad
        integer entidad_id
        timestamp timestamp
        text patch_json
        text autor
        boolean revertido
    }

    FAMILIA_ANONIMIZADA {
        text uuid PK
        integer comunidad_id FK
        integer num_personas
        text estrato
        text material_paredes
        text material_piso
        text material_techo
        boolean acceso_agua
        boolean acceso_energia
        boolean acceso_alcantarillado
        text tipo_tenencia
        boolean hacinamiento
        float score_prioridad
    }

    COMUNIDAD ||--o{ FAMILIA : "tiene"
    FAMILIA ||--o| CARACTERIZACION : "tiene (encuesta inicial)"
    FAMILIA ||--o{ ENTREVISTA_SEGUIMIENTO : "tiene (seguimientos)"
    FAMILIA ||--o{ PATCH_LOG : "auditada en"
    FAMILIA }o--o| FAMILIA : "duplicado_de"
    COMUNIDAD ||--o{ FAMILIA_ANONIMIZADA : "tiene"
```

### Descripción de Entidades

| Entidad | Descripción | Fuente ODS | Base de datos |
|---|---|---|---|
| `COMUNIDAD` | Las 3 comunidades del proyecto (Granizal, La Honda, Nueva Jerusalen) | — | `caracterizaciones_limpio.db` |
| `FAMILIA` | Registro de cada hogar con datos completos incluyendo PII | Ambas pestañas | `caracterizaciones_limpio.db` |
| `CARACTERIZACION` | Encuesta inicial vía Google Form: condiciones de vivienda, servicios, hacinamiento | Pestaña "Caracterizaciones" (Sheet 1) | `caracterizaciones_limpio.db` |
| `ENTREVISTA_SEGUIMIENTO` | Visitas de seguimiento por comunidad, post-encuesta inicial | Pestañas por comunidad (Sheet 2) | `caracterizaciones_limpio.db` |
| `PATCH_LOG` | Historial de cambios vía API — cada PATCH genera un JSON Patch (RFC 6902) | — | `caracterizaciones_limpio.db` |
| `FAMILIA_ANONIMIZADA` | Versión sin PII de `FAMILIA` + `CARACTERIZACION` para app móvil, con UUID | — | `comunidad_anonimizada.db` |

### Campos PII (requieren anonimización)

`nombre_jefe_hogar`, `cedula`, `telefono`, `direccion` — eliminados o hasheados en `FAMILIA_ANONIMIZADA`.

---

## Diseño del Sistema a Alto Nivel

El sistema se organiza en tres capas: **ingesta/transformación** (notebooks), **almacenamiento** (SQLite), y **exposición** (FastAPI).

```mermaid
flowchart TD
    subgraph Fuente["Fuente de Datos"]
        ODS["📄 in/sheet.ods\n(4 pestañas)"]
    end

    subgraph Pipeline["Pipeline — Jupyter Notebooks (nbclick)"]
        N1["01_ingesta\nODS → SQLite crudo"]
        N2["02_analisis\nEstadísticas descriptivas"]
        N3["03_limpieza\nNormalización + dedup"]
        N4["04_anonimizacion\nEliminar PII + UUIDs"]
    end

    subgraph Storage["Almacenamiento — SQLite"]
        DBr["caracterizaciones.db\n(crudo, solo lectura)"]
        DBl["caracterizaciones_limpio.db\n(normalizado + patches)"]
        DBa["comunidad_anonimizada.db\n(sin PII)"]
        REP["reporte_limpieza.html"]
    end

    subgraph API["Backend — FastAPI"]
        R1["GET /comunidades\n(lectura paginada)"]
        R2["GET /comunidades?q=\n(búsqueda)"]
        R3["PATCH /caracterizaciones/:id\n(escritura + patch log)"]
        R4["GET /historial/:id\n(auditoría)"]
    end

    subgraph Clientes["Clientes"]
        APP["📱 App Móvil\n(React Native)"]
        COORD["👤 Coordinador\n(browser / CLI)"]
    end

    ODS --> N1
    N1 --> DBr
    DBr --> N2
    N2 --> N3
    N3 --> DBl
    N3 --> REP
    DBl --> N4
    N4 --> DBa

    DBa --> R1
    DBa --> R2
    DBl --> R3
    DBl --> R4

    R1 --> APP
    R2 --> APP
    R3 --> COORD
    R4 --> COORD
```

### Principios de Diseño

- **Inmutabilidad por capas**: cada fase lee de la anterior pero nunca la modifica. `caracterizaciones.db` es la fuente de verdad cruda, siempre preservada.
- **Idempotencia**: re-ejecutar cualquier notebook produce el mismo resultado dado el mismo input.
- **Separación PII / no-PII**: los datos personales nunca salen de `caracterizaciones_limpio.db`. La API pública solo toca `comunidad_anonimizada.db`.
- **Trazabilidad**: toda escritura vía API genera un JSON Patch almacenado e individualmente reversible.

---

## Diagrama C4 — Zoom en el Backend API

### Nivel 1 — Contexto del Sistema

```mermaid
C4Context
    title Sistema Techo Cleanup — Contexto

    Person(coord, "Coordinador de Datos", "Ejecuta el pipeline, actualiza registros, revisa historial")
    Person(vol, "Voluntario de Campo", "Consulta datos de comunidad desde la app móvil")
    Person(dev, "Desarrollador App", "Integra la API en la aplicación React Native")

    System(techo, "Techo Cleanup", "Pipeline de datos + API REST para datos de caracterizaciones comunitarias")

    System_Ext(ods, "Planilla ODS", "Fuente de datos de campo (Google Sheets / LibreOffice)")
    System_Ext(app, "App Móvil", "React Native / Expo — toma y consulta de datos en campo")

    Rel(coord, techo, "Ejecuta notebooks, hace PATCH a registros")
    Rel(vol, app, "Usa para consultar familias")
    Rel(dev, techo, "Integra vía API REST")
    Rel(app, techo, "GET /comunidades, búsqueda")
    Rel(techo, ods, "Lee (solo lectura)")
```

### Nivel 2 — Contenedores

```mermaid
C4Container
    title Techo Cleanup — Contenedores

    Person(coord, "Coordinador")
    Person(vol, "Voluntario / App")

    Container(nb, "Pipeline Notebooks", "Python / Jupyter / nbclick", "Ingesta, limpieza y anonimización de datos. Ejecutado manualmente o por cron.")
    Container(api, "FastAPI Backend", "Python / FastAPI / Uvicorn", "API REST. Sirve datos anonimizados (lectura) y datos internos con patch log (escritura).")
    ContainerDb(dbl, "DB Interna", "SQLite", "caracterizaciones_limpio.db — familias, caracterizaciones, patches")
    ContainerDb(dba, "DB Anonimizada", "SQLite", "comunidad_anonimizada.db — sin PII, solo campos agregados")

    Rel(coord, nb, "Ejecuta via CLI / nbclick")
    Rel(nb, dbl, "Escribe (normalizado + patches)")
    Rel(nb, dba, "Escribe (anonimizado)")
    Rel(vol, api, "GET /comunidades, búsqueda")
    Rel(coord, api, "PATCH /caracterizaciones, GET /historial")
    Rel(api, dba, "Lee (endpoints públicos)")
    Rel(api, dbl, "Lee/Escribe (endpoints internos + patch log)")
```

### Nivel 3 — Componentes del Backend API

```mermaid
C4Component
    title FastAPI Backend — Componentes Internos

    Container_Boundary(api, "FastAPI Backend") {
        Component(router_pub, "Router Público", "FastAPI APIRouter", "GET /comunidades, GET /comunidades/{id}, GET /comunidades?q=. Solo lectura sobre DB anonimizada.")
        Component(router_int, "Router Interno", "FastAPI APIRouter", "POST/PATCH /caracterizaciones, GET/DELETE /historial. Requiere auth header (v2).")
        Component(patch_svc, "PatchService", "Python class", "Genera JSON Patch (RFC 6902) comparando estado anterior vs nuevo. Persiste en tabla patches. Gestiona reversión.")
        Component(anon_repo, "AnonRepo", "SQLAlchemy / raw sqlite3", "Queries sobre comunidad_anonimizada.db. Paginación, FTS, filtro por comunidad.")
        Component(intern_repo, "InternRepo", "SQLAlchemy / raw sqlite3", "CRUD sobre caracterizaciones_limpio.db. Incluye joins con patches.")
        Component(schemas, "Schemas Pydantic", "Pydantic v2", "Modelos de request/response. Valida entrada, serializa salida. Garantiza que PII no escape en respuestas públicas.")
    }

    ContainerDb(dba, "DB Anonimizada", "SQLite")
    ContainerDb(dbl, "DB Interna", "SQLite")
    Person(vol, "Voluntario / App")
    Person(coord, "Coordinador")

    Rel(vol, router_pub, "HTTP GET")
    Rel(coord, router_int, "HTTP PATCH / GET / DELETE")
    Rel(router_pub, schemas, "Valida respuesta")
    Rel(router_pub, anon_repo, "Consulta datos")
    Rel(router_int, schemas, "Valida request y respuesta")
    Rel(router_int, intern_repo, "Lee/escribe registro")
    Rel(router_int, patch_svc, "Delega generación de patch")
    Rel(anon_repo, dba, "SQL queries")
    Rel(intern_repo, dbl, "SQL queries")
    Rel(patch_svc, dbl, "INSERT INTO patches")
```

### Nivel 4 — Detalle: PatchService

```mermaid
flowchart TD
    A["PATCH /caracterizaciones/42\n{num_personas: 5}"] --> B["InternRepo.get(42)\nLee estado actual del registro"]
    B --> C["Compara estado_anterior vs payload\nusing jsonpatch.make_patch()"]
    C --> D{"¿Hay diferencias?"}
    D -- No --> E["Return 304 Not Modified"]
    D -- Sí --> F["InternRepo.update(42, payload)\nAplica cambios en DB"]
    F --> G["INSERT INTO patches\n(entidad='familia', entidad_id=42,\npatch_json=..., autor=req.autor,\ntimestamp=now(), revertido=False)"]
    G --> H["Return 200 OK\n+ patch_id generado"]

    style A fill:#e8f4f8
    style H fill:#e8f8e8
    style E fill:#f8f4e8
```
