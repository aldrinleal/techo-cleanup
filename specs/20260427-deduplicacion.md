# Spec: Fase 2 — Deduplicación de Familias

**Fecha**: 2026-04-27  
**Estado**: Propuesta  
**Notebook objetivo**: `notebooks/02_deduplicacion.ipynb` (`pipenv run nbclick notebooks/02_deduplicacion.ipynb`)  
**Dependencia**: `notebooks/01_ingesta.ipynb` (Fase 1 completada)

---

## 1. Problema

Los datos crudos importados en Fase 1 contienen duplicados de al menos dos orígenes:

| Tipo | Origen | Ejemplo |
|---|---|---|
| **Intra-sheet** | Una familia llena el Google Form más de una vez | Mismo nombre/cédula en "Caracterizaciones" dos veces |
| **Intra-comunidad** | Doble ingreso manual en una pestaña de comunidad | Mismo hogar en "Granizal" con dos fechas distintas |
| **Cross-comunidad** | Familia registrada en dos comunidades distintas | Caso real: familia que se mudó, quedó en ambas hojas |

El reto es que los datos de campo tienen **ruido real**: cédulas con dígitos transpuestos, nombres con o sin tildes, direcciones abreviadas de distintas formas. La deduplicación pura por igualdad exacta es insuficiente.

---

## 2. Alcance

**Dentro del alcance**:
- Deduplicación dentro de `caracterizaciones` (pestaña Caracterizaciones)
- Deduplicación dentro de cada comunidad (intra-sheet por comunidad)
- Detección opcional de duplicados cross-comunidad

**Fuera del alcance en esta fase**:
- El join deliberado Caracterizaciones ↔ comunidades (relación intencional, no es un duplicado)
- Limpieza de texto general (trim, UTF-8) — se maneja en Fase 3 (`03_limpieza.ipynb`)
- Eliminación de registros — los duplicados **se marcan, nunca se eliminan**

---

## 3. Campos Identificadores

Basado en la estructura del ODS:

| Campo | Rol | Calidad esperada |
|---|---|---|
| `cedula` | Identificador primario | Alta confianza pero con typos frecuentes |
| `nombre_jefe_hogar` | Identificador secundario | Variaciones ortográficas, apodos |
| `telefono` | Señal de apoyo | No único (líneas compartidas), muchos nulos |
| `direccion` | Señal de apoyo | Alta variabilidad de formato |
| `comunidad` | Partición | Reduce el espacio de búsqueda |

---

## 4. Algoritmo Multi-Paso

El pipeline ejecuta los pasos en orden. El primero que clasifica una pareja como duplicado gana; los demás no se aplican a esa pareja.

```
Paso 1 → Exacto por cédula
Paso 2 → Fuzzy por cédula (rapidfuzz)
Paso 3 → Fuzzy por nombre + dirección (rapidfuzz)
Paso 4 → Revisión LLM (opcional, solo casos inciertos)
```

### Paso 1 — Exacto por cédula

```python
# Condición de duplicado
GROUP BY cedula HAVING COUNT(*) > 1
```

- Se aplica por separado dentro de `caracterizaciones` y dentro de cada comunidad
- Si `cedula` es nulo o vacío, el registro pasa al paso siguiente
- Resultado: confianza **alta**, no requiere revisión humana

### Paso 2 — Fuzzy por cédula

Usa `rapidfuzz.fuzz.ratio` sobre la cadena de cédula normalizada (solo dígitos).

```python
cedula_norm = re.sub(r'\D', '', str(cedula))
```

| Score | Decisión |
|---|---|
| ≥ 95 | Duplicado probable → flag automático |
| 80–94 | Incierto → cola para revisión (LLM o manual) |
| < 80 | No duplicado |

Umbrales configurables por parámetro en el notebook.

### Paso 3 — Fuzzy por nombre + dirección

Combina dos scores con peso configurable (default: 70% nombre, 30% dirección):

```python
score = 0.7 * fuzz.token_sort_ratio(nombre_a, nombre_b) \
      + 0.3 * fuzz.token_sort_ratio(dir_a, dir_b)
```

`token_sort_ratio` es preferido a `ratio` porque maneja el orden de palabras ("García Juan" vs "Juan García").

| Score combinado | Decisión |
|---|---|
| ≥ 88 | Duplicado probable |
| 70–87 | Incierto → cola para revisión |
| < 70 | No duplicado |

### Paso 4 — Revisión LLM (opcional)

Aplica únicamente a los pares en la zona incierta de los pasos 2 y 3.

**Activación**: parámetro `--llm` en nbclick o variable `LLM_DEDUP_ENABLED=true` en `.env`.

---

## 5. Diseño de la Integración LLM

### 5.1 Motivación

El matching fuzzy clásico falla en casos como:

| Caso | Ejemplo | Problema |
|---|---|---|
| Abreviaciones de dirección | "Cra 15" vs "Carrera 15" | Score bajo por tokens distintos |
| Apodos / variantes del nombre | "Consuelo" vs "Chelo Martínez" | Sin parecido léxico |
| Cédulas con dígito extra | "1023456789" vs "01023456789" | Fuzzy da 92%, no 100% |
| Datos contradictorios | Mismo nombre, distinto teléfono, misma dirección | ¿Error de campo o persona distinta? |

El LLM aporta razonamiento contextual que los algoritmos deterministas no pueden proveer.

### 5.2 Arquitectura

```
Pares inciertos
      │
      ▼
┌─────────────────────────┐
│  Cache lookup           │  ← hash(id_a, id_b) → resultado previo
└─────────────────────────┘
      │ miss
      ▼
┌─────────────────────────┐
│  Redactar prompt        │  ← campos no-PII primero (ver 5.3)
└─────────────────────────┘
      │
      ▼
┌─────────────────────────┐
│  LLM via LangChain      │  ← modelo configurable, default: gpt-4o-mini
│  (OpenRouter o directo) │
└─────────────────────────┘
      │
      ▼
┌─────────────────────────┐
│  Parsear JSON response  │  ← con retry en fallo de parse
└─────────────────────────┘
      │
      ▼
  Resultado almacenado en cache + DB
```

### 5.3 Privacidad: qué enviar al LLM

> **Decisión requerida del equipo antes de implementar**: ¿aceptan que los datos de familias salgan a una API externa?

Tres modos disponibles, configurables en `.env`:

| Modo | `LLM_PRIVACY_MODE` | Qué se envía | Riesgo PII |
|---|---|---|---|
| **full** (default dev) | `full` | Todos los campos | Alto — solo usar con modelos locales o en dev |
| **partial** | `partial` | Solo campos no-PII: comunidad, num_personas, material_vivienda, scores fuzzy | Bajo |
| **scores_only** | `scores_only` | Solo los scores de similitud + metadatos estadísticos | Ninguno |

Para producción con datos reales, **se recomienda `partial`** como mínimo.

### 5.4 Prompt del LLM

```
Sistema:
Eres un asistente de calidad de datos para una ONG que trabaja con familias
vulnerables. Tu tarea es determinar si dos registros de familias representan
el mismo hogar o son hogares distintos.

Responde SOLO con un JSON válido con esta estructura exacta:
{
  "es_duplicado": true | false,
  "confianza": 0.0 a 1.0,
  "razonamiento": "explicación breve en español"
}

Usuario:
Registro A:
{{campos_registro_a}}

Registro B:
{{campos_registro_b}}

Scores de similitud calculados previamente:
- Similitud de cédula: {{score_cedula}}
- Similitud de nombre: {{score_nombre}}
- Similitud de dirección: {{score_direccion}}

¿Son el mismo hogar?
```

### 5.5 Control de Costos

- **Batching**: procesar en lotes de 20 pares por llamada (un par por mensaje de usuario en un thread de conversación)
- **Presupuesto máximo**: configurable `LLM_MAX_PAIRS=500` (default) — abortará si hay más pares inciertos sin procesar
- **Cache persistente**: `out/llm_dedup_cache.json` — evita re-llamar al LLM en re-ejecuciones
- **Estimado de costo**: ~$0.002 por par con gpt-4o-mini (500 pares ≈ $1 USD)

---

## 6. Esquema de Resultados

Nuevas columnas añadidas a las tablas existentes (sin modificar `caracterizaciones.db`):

```sql
-- En caracterizaciones_dedup.db (nuevo archivo, no modifica Fase 1)
ALTER TABLE caracterizaciones ADD COLUMN es_duplicado BOOLEAN DEFAULT FALSE;
ALTER TABLE caracterizaciones ADD COLUMN duplicado_de INTEGER REFERENCES caracterizaciones(id);
ALTER TABLE caracterizaciones ADD COLUMN dedup_metodo TEXT;  -- 'exact_cedula', 'fuzzy_cedula', 'fuzzy_nombre', 'llm'
ALTER TABLE caracterizaciones ADD COLUMN dedup_score REAL;   -- 0.0–1.0
ALTER TABLE caracterizaciones ADD COLUMN dedup_confianza TEXT; -- 'high', 'medium', 'low'
ALTER TABLE caracterizaciones ADD COLUMN dedup_razonamiento TEXT; -- solo si vino de LLM
```

El mismo esquema aplica para `comunidades_dedup.db`.

**Convención de "master"**: cuando dos registros son duplicados, el más antiguo (menor timestamp de entrada) se marca como master (`es_duplicado=False`); el otro lleva `es_duplicado=True` y `duplicado_de=<id_master>`.

---

## 7. Outputs del Notebook

| Artefacto | Ruta | Descripción |
|---|---|---|
| DB deduplicada | `out/caracterizaciones_dedup.db` | Caracterizaciones con flags de duplicados |
| DB deduplicada | `out/comunidades_dedup.db` | Comunidades con flags de duplicados |
| Cache LLM | `out/llm_dedup_cache.json` | Resultados LLM cacheados (no commitear) |
| Reporte | `out/reporte_dedup.html` | Resumen visual de duplicados encontrados |

### Contenido del reporte HTML

- Tabla resumen: total de registros, duplicados encontrados por método, % de cobertura
- Por comunidad: breakdown de duplicados intra-comunidad
- Lista de pares inciertos sin resolver (si los hay)
- Estimado de costo LLM (si se usó)

---

## 8. Criterios de Aceptación

- [ ] `notebooks/02_deduplicacion.ipynb` ejecutable con `pipenv run nbclick notebooks/02_deduplicacion.ipynb`
- [ ] Paso 1 (exacto por cédula) implementado y testeado con datos sintéticos en el notebook
- [ ] Paso 2 (fuzzy cédula) implementado con umbrales configurables
- [ ] Paso 3 (fuzzy nombre+dirección) implementado con pesos configurables
- [ ] Paso 4 (LLM) implementado pero desactivado por defecto (`LLM_DEDUP_ENABLED=false`)
- [ ] Los duplicados se marcan con `es_duplicado=True` y `duplicado_de=<id>` — **nunca se eliminan**
- [ ] El notebook es idempotente: re-ejecutar produce el mismo resultado
- [ ] `out/reporte_dedup.html` generado con conteos por método y por comunidad
- [ ] Los archivos de Fase 1 (`caracterizaciones.db`, `comunidades.db`) no se modifican
- [ ] Cache LLM (`out/llm_dedup_cache.json`) en `.gitignore`
- [ ] Log al final de la ejecución: `N pares procesados, M duplicados encontrados (X exact, Y fuzzy, Z llm)`
- [ ] Si `LLM_DEDUP_ENABLED=true` pero las variables de API no están seteadas, el notebook falla con mensaje claro

---

## 9. Decisiones Abiertas

| # | Decisión | Opciones | Recomendación |
|---|---|---|---|
| D-01 | ¿Qué datos se envían al LLM? | `full`, `partial`, `scores_only` | `partial` para producción |
| D-02 | ¿Modelo LLM? | gpt-4o-mini (barato), Claude Haiku (disponible vía OpenRouter), Ollama local | gpt-4o-mini vía OpenRouter como default |
| D-03 | ¿Dedup cross-comunidad en esta fase? | Sí / No | No — agregaría complejidad; hacerlo en Fase 3 limpieza |
| D-04 | ¿Umbral de confianza para auto-flag sin revisión? | Configurable vs fijo | Configurable con defaults razonables en `.env.sample` |

---

## 10. Variables de Entorno Nuevas

Agregar a `.env.sample`:

```bash
# Deduplicación — Fase 2
LLM_DEDUP_ENABLED=false           # true para activar revisión LLM de pares inciertos
LLM_PRIVACY_MODE=partial          # full | partial | scores_only
LLM_MAX_PAIRS=500                 # límite de pares a enviar al LLM por ejecución
DEDUP_FUZZY_CEDULA_HIGH=95        # score >= este valor → duplicado automático (cédula fuzzy)
DEDUP_FUZZY_CEDULA_LOW=80         # score < este valor → no duplicado (cédula fuzzy)
DEDUP_FUZZY_NOMBRE_HIGH=88        # idem para nombre+dirección
DEDUP_FUZZY_NOMBRE_LOW=70         # idem para nombre+dirección
```

---

## 11. Estimado

**5 puntos Fibonacci** (sin LLM: 3 puntos; con LLM: +2 puntos adicionales)

Desglose:
- Infraestructura del notebook + lectura de DBs: 0.5
- Paso 1 exacto: 0.5
- Paso 2 fuzzy cédula: 1.0
- Paso 3 fuzzy nombre+dir: 1.0
- Reporte HTML: 0.5
- Paso 4 LLM (opcional): 2.0
- Cache LLM + control de costos: incluido en el paso 4
