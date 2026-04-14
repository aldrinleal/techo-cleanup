# Contexto: TECHO Colombia — Pipeline de Limpieza y Gestión de Datos

**Fecha**: 2026-04-13
**Organización**: TECHO Colombia (voluntarios)
**Proyecto**: Pipeline de limpieza, transformación y exposición de datos de caracterizaciones comunitarias

---

## 1. Contexto Organizacional

TECHO es una organización sin fines de lucro que trabaja con comunidades en asentamientos informales. Los voluntarios realizan **caracterizaciones** (entrevistas a familias) para documentar las condiciones habitacionales, socioeconómicas y de acceso a servicios de cada hogar.

Los datos se recolectan en campo y se consolidan en planillas ODS (LibreOffice Calc). Este proyecto busca transformar esas planillas en una base de datos limpia, estructurada y utilizable para toma de decisiones y aplicaciones móviles de campo.

---

## 2. Fuente de Datos

**Archivo principal**: `in/sheet.ods`

### Pestañas relevantes

| Pestaña | Contenido |
|---|---|
| Caracterizaciones | Entrevistas individuales a familias (datos maestros) |
| Vereda El Granizal | Datos específicos de la comunidad El Granizal |
| Manrique La Honda | Datos específicos de la comunidad Manrique La Honda |
| Vereda La Nueva Jerusalen | Datos específicos de la comunidad Vereda La Nueva Jerusalen |

---

## 3. Product Requirements Document (PRD)

### Visión del Producto

Un pipeline reproducible y automatizable que transforma planillas ODS con datos crudos de campo en:
1. Una base de datos SQLite limpia y normalizada (uso interno)
2. Una versión anonimizada para uso en aplicaciones móviles
3. Una API backend que sirve los datos anonimizados y acepta actualizaciones con trazabilidad

---

### Non-Goals (Fuera de Alcance)

- No construir la aplicación móvil (solo el backend/API que ella consumirá)
- No reemplazar el proceso de recolección de datos en campo
- No implementar autenticación de usuarios ni roles de acceso (v1)
- No migrar a una base de datos en la nube (SQLite local es suficiente para v1)
- No procesar otras planillas fuera de `in/sheet.ods`
- No construir dashboards ni visualizaciones (fuera del análisis exploratorio en notebooks)

---

### Stakeholders

- **Voluntarios de campo**: generan los datos en ODS
- **Coordinadores de comunidad**: necesitan reportes limpios y sin duplicados
- **Desarrolladores de app móvil**: consumen la API anonimizada
- **Equipo de datos TECHO**: ejecutan el pipeline periódicamente

---

## 4. Fases del Proyecto

---

### Fase 1 — Ingesta y Conversión ODS → SQLite

**Dependencias**: Ninguna (fase inicial)

**Objetivo**: Leer `in/sheet.ods` y producir dos archivos SQLite en `out/`:
- `out/caracterizaciones.db` — datos de la pestaña Caracterizaciones
- `out/comunidades.db` — datos unificados de las tres pestañas de comunidades

**Criterios de Aceptación**:
- [ ] Script `notebooks/01_ingesta.ipynb` ejecutable con `nbclick run`
- [ ] Produce `out/caracterizaciones.db` con tabla `caracterizaciones`
- [ ] Produce `out/comunidades.db` con tabla `comunidades` que incluye columna `comunidad` para identificar la fuente
- [ ] Todas las columnas del ODS se preservan (sin pérdida de datos en esta fase)
- [ ] El script es idempotente (re-ejecutar produce el mismo resultado)
- [ ] Log de ejecución indica cuántas filas se importaron por pestaña

**Qué NO debe cambiar**:
- Los datos originales del ODS (solo lectura)
- El esquema original de las columnas (no normalizar aún)

**Estimado**: 3 puntos (Fibonacci)

---

### Fase 2 — Análisis y Modelado de Datos

**Dependencias**: Fase 1

**Objetivo**: Documentar el modelo de datos real, identificar tipos, valores nulos, inconsistencias y definir el esquema normalizado.

**Criterios de Aceptación**:
- [ ] `notebooks/02_analisis_exploratorio.ipynb` con estadísticas descriptivas de cada columna
- [ ] `specs/modelo-datos.md` con diagrama de entidades y descripción de cada campo
- [ ] `specs/arquitectura.md` con diagrama C4 del sistema completo y diagrama de arquitectura del pipeline
- [ ] Lista de columnas con >20% de valores nulos documentada
- [ ] Identificación de posibles campos de identidad personal (PII) para anonimización

**Qué NO debe cambiar**:
- Los archivos SQLite de Fase 1 (este análisis es solo lectura)

**Estimado**: 5 puntos

---

### Fase 3 — Limpieza y Deduplicación

**Dependencias**: Fase 1, Fase 2

**Objetivo**: Aplicar reglas de limpieza sobre los datos y detectar/marcar duplicados.

**Criterios de Aceptación**:
- [ ] `notebooks/03_limpieza.ipynb` con reglas de normalización documentadas
- [ ] Columnas de texto: trim, lowercase donde aplique, codificación UTF-8 consistente
- [ ] Fechas: estandarizadas a ISO 8601
- [ ] Script de detección de duplicados con umbral configurable (fuzzy matching en nombre + dirección)
- [ ] Los duplicados se **marcan** con flag `es_duplicado=True` y `duplicado_de=<id>` — no se eliminan
- [ ] `out/caracterizaciones_limpio.db` y `out/comunidades_limpio.db` producidos
- [ ] Reporte HTML generado en `out/reporte_limpieza.html` con conteos antes/después

**Qué NO debe cambiar**:
- Los archivos SQLite de Fase 1 (fuente de verdad original se preserva)

**Estimado**: 8 puntos

---

### Fase 4 — Exportación Anonimizada

**Dependencias**: Fase 3

**Objetivo**: Generar una versión de los datos sin PII para uso en aplicaciones móviles.

**Criterios de Aceptación**:
- [ ] `notebooks/04_anonimizacion.ipynb` con reglas de anonimización documentadas
- [ ] Campos PII (nombres, documentos, teléfonos, direcciones exactas) eliminados o hasheados
- [ ] IDs internos reemplazados por UUIDs aleatorios (no correlacionables con el original)
- [ ] `out/comunidad_anonimizada.db` producido
- [ ] Test: ninguna fila del output contiene nombre, documento o teléfono real
- [ ] Script ejecutable desde CLI: `nbclick run notebooks/04_anonimizacion.ipynb`

**Qué NO debe cambiar**:
- Los archivos SQLite de Fase 3 (fuente limpia se preserva)
- Las reglas de anonimización son deterministas dado el mismo input

**Estimado**: 5 puntos

---

### Fase 5 — API Backend (Datos Anonimizados)

**Dependencias**: Fase 4

**Objetivo**: FastAPI que sirve `out/comunidad_anonimizada.db` con endpoints REST de lectura.

**Criterios de Aceptación**:
- [ ] `src/api/main.py` con FastAPI app
- [ ] `GET /comunidades` — lista todas las comunidades
- [ ] `GET /comunidades/{id}` — detalle de una entrada
- [ ] `GET /comunidades?q=<texto>` — búsqueda por texto
- [ ] Paginación con `limit` y `offset`
- [ ] OpenAPI docs en `/docs`
- [ ] Dockerfile o script de inicio documentado
- [ ] Tests de integración básicos con `pytest`

**Qué NO debe cambiar**:
- La base anonimizada es de solo lectura vía esta API en v1

**Estimado**: 8 puntos

---

### Fase 6 — API con Escritura y JSON Patches (Opcional)

**Dependencias**: Fase 5

**Objetivo**: Permitir operaciones REST sobre la base de datos no anonimizada, almacenando cada cambio como un JSON Patch (RFC 6902) para trazabilidad completa.

**Criterios de Aceptación**:
- [ ] `POST /caracterizaciones` — crear nueva entrada
- [ ] `PATCH /caracterizaciones/{id}` — actualizar parcialmente, genera JSON Patch y lo almacena
- [ ] `GET /caracterizaciones/{id}/historial` — devuelve lista de patches aplicados
- [ ] Tabla `patches` en SQLite: `id`, `entidad`, `entidad_id`, `timestamp`, `patch_json`, `autor`
- [ ] Reversión de patch individual: `DELETE /caracterizaciones/{id}/historial/{patch_id}`
- [ ] Tests de integración para flujo completo CRUD + historial

**Qué NO debe cambiar**:
- La API de Fase 5 (compatibilidad backward garantizada)

**Estimado**: 13 puntos

---

## 5. User Stories

### Pipeline de Datos

- **US-01**: Como coordinador de datos, quiero ejecutar un solo comando para regenerar las bases SQLite desde el ODS, para que el proceso sea reproducible sin intervención manual.
- **US-02**: Como coordinador, quiero ver un reporte de cuántas filas se importaron por comunidad, para verificar que no se perdieron datos.
- **US-03**: Como analista, quiero un notebook con estadísticas descriptivas de los datos crudos, para entender la calidad antes de limpiar.
- **US-04**: Como coordinador, quiero que los duplicados sean marcados pero no eliminados, para poder auditarlos manualmente si hay dudas.
- **US-05**: Como coordinador, quiero un reporte HTML del proceso de limpieza, para compartirlo con el equipo sin que accedan a la base.

### Anonimización

- **US-06**: Como desarrollador de la app móvil, quiero una base SQLite sin datos personales, para poder distribuirla sin riesgos de privacidad.
- **US-07**: Como coordinador, quiero que la anonimización sea reproducible, para poder regenerar la base en cualquier momento.

### API

- **US-08**: Como desarrollador de app, quiero un endpoint REST para listar comunidades con paginación, para consumirlo eficientemente desde mobile.
- **US-09**: Como desarrollador de app, quiero buscar por texto libre en los datos de comunidad, para encontrar registros sin conocer el ID exacto.
- **US-10**: Como coordinador, quiero que cada cambio a un registro genere un JSON Patch almacenado, para tener historial completo de modificaciones.
- **US-11**: Como auditor, quiero ver el historial de patches de un registro, para entender cómo evolucionaron los datos.

---

## 6. Backlog Priorizado

| Prioridad | Fase | Historia | Estimado (Fibonacci) | Dependencia |
|---|---|---|---|---|
| 1 | Fase 1 | Ingesta ODS → SQLite | 3 | — |
| 2 | Fase 2 | Análisis y modelado | 5 | Fase 1 |
| 3 | Fase 3 | Limpieza y deduplicación | 8 | Fase 2 |
| 4 | Fase 4 | Exportación anonimizada | 5 | Fase 3 |
| 5 | Fase 5 | API REST lectura | 8 | Fase 4 |
| 6 | Fase 6 | API con JSON Patches | 13 | Fase 5 |
| **Total** | | | **42 puntos** | |

---

## 7. Stack Técnico

| Componente | Tecnología |
|---|---|
| Runtime | Python 3.11+, Pipenv, pyproject.toml |
| Notebooks | Jupyter, nbclick (CLI para notebooks) |
| LLM | LangChain + OpenRouter |
| Base de datos | SQLite3 |
| API | FastAPI + Uvicorn |
| Storage | S3 o compatible (artefactos) |
| Testing | pytest, pytest-asyncio |
| Datos fuente | LibreOffice ODS (`.ods`) |

---

## 8. Riesgos

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| Estructura del ODS cambia entre versiones | Media | Alto | Validar schema en Fase 1 con tests |
| Falsos positivos en deduplicación | Alta | Medio | Marcar, no eliminar; revisión manual |
| PII residual en base anonimizada | Baja | Muy alto | Tests explícitos en Fase 4 |
| Performance de API con DB grande | Baja | Medio | Índices en SQLite desde Fase 5 |
