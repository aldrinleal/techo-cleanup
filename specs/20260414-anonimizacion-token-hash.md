# Spec: Anonimización por Hash de Tokens

**Fecha**: 2026-04-14
**Fase asociada**: Fase 4 — Exportación Anonimizada
**Estado**: Aprobada — pendiente implementación

---

## 1. Motivación

El enfoque original de Fase 4 planteaba **eliminar** los campos PII del output.
Ese enfoque tiene un problema crítico: **destruye la capacidad de cruzar datos**.

Si se elimina el campo `cedula`, no se puede:
- Detectar duplicados en la base anonimizada
- Cruzar `caracterizaciones` con `comunidades` en el output público
- Verificar si una familia ya fue atendida sin exponer el número real

El nuevo enfoque reemplaza cada **token** de un campo sensible por su hash determinista.
Esto preserva estructura y permite cruce sin revelar el valor original.

---

## 2. Algoritmo

### Función `anon_token`

```python
import hashlib
import unicodedata

def anon_token(s: str) -> str:
    """Convierte un token en su hash pseudoanónimo de 8 caracteres hex."""
    # 1. Normalizar unicode y eliminar acentos
    s = unicodedata.normalize("NFD", s)
    s = "".join(c for c in s if unicodedata.category(c) != "Mn")
    # 2. Minúsculas
    s = s.lower()
    # 3. SHA1 → tomar los primeros 8 caracteres del hexdigest
    return hashlib.sha1(s.encode("utf-8")).hexdigest()[:8]
```

> **Nota**: se usa `unicodedata` (stdlib) en lugar de `unidecode` (tercero) para no agregar dependencias innecesarias. El resultado es equivalente para nombres en español.

### Función `anon_field`

```python
def anon_field(value: str | None) -> str | None:
    """Tokeniza un campo y hashea cada token."""
    if value is None or str(value).strip() == "":
        return value
    tokens = str(value).split()
    return " ".join(anon_token(t) for t in tokens)
```

---

## 3. Propiedades del Algoritmo

| Propiedad | Valor |
|---|---|
| Determinista | Sí — mismo input → mismo hash siempre |
| Reversible | No (en práctica) — preimage de SHA1 sobre tokens cortos es computacionalmente costoso |
| Preserva estructura | Sí — número de tokens, presencia de campo, nulls |
| Permite JOIN entre tablas | Sí — `anon_field(cedula)` es igual en ambas tablas si el valor original es el mismo |
| Permite deduplicación | Sí — registros con mismo nombre/cédula coinciden en el hash |
| Resistente a diccionario | Parcialmente — nombres comunes en español podrían ser adivinados por fuerza bruta |

### Sobre resistencia a fuerza bruta

Este algoritmo implementa **seudonimización**, no anonimización fuerte (GDPR Art. 4(5)).
Con ~10K nombres frecuentes en Colombia, un atacante podría recuperar nombres comunes.
Para el caso de uso de TECHO (distribución interna entre voluntarios, no publicación pública),
este nivel de protección es **suficiente** y proporcional al riesgo documentado en el PRD.

Si en el futuro se requiere anonimización fuerte, se puede agregar un **salt secreto** al hash:
```python
SALT = os.environ.get("ANON_SALT", "")
hashlib.sha1((SALT + s).encode("utf-8")).hexdigest()[:8]
```

Con salt, el hash no es reproducible por externos aunque conozcan el algoritmo.
Esta mejora se deja como extensión opcional (ver sección 6).

---

## 4. Campos a Anonimizar

Los campos PII conocidos a partir de la Fase 1 son:

### Tabla `caracterizaciones`

| Campo original | Tipo PII | Tratamiento |
|---|---|---|
| `Nombre del interesado` (o similar) | Nombre completo | `anon_field` |
| `Documento de identificación del interesado` | Cédula | `anon_field` (token único) |
| `Número de celular` / `Teléfono` | Teléfono | `anon_field` (token único) |
| `Dirección` | Dirección exacta | `anon_field` |
| `Correo electrónico` | Email | `anon_field` (token único) |

### Tabla `comunidades`

| Campo original | Tipo PII | Tratamiento |
|---|---|---|
| `NOMBRE` / `NOMBRES` | Nombre completo | `anon_field` |
| `CEDULA` | Cédula | `anon_field` (token único) |
| `CELULAR` / `TELEFONO` | Teléfono | `anon_field` (token único) |
| `DIRECCION` | Dirección | `anon_field` |

> **Nota**: La lista exacta se confirma en Fase 2 (análisis exploratorio). El notebook
> `04_anonimizacion.ipynb` debe tener una sección de configuración con la lista de columnas
> PII explícita y modificable.

### Campos NO anonimizados

Todo lo que no sea PII directo permanece legible:
- Comunidad / barrio (granularidad geográfica aceptable)
- Fechas de encuesta
- Variables ordinales (nivel socioeconómico, tipo de vivienda, etc.)
- Flags de estado (asignado, duplicado, etc.)

---

## 5. Criterios de Aceptación (actualización Fase 4)

Reemplaza los criterios originales de Fase 4 en el PRD:

- [ ] Función `anon_token(s)` implementada y con tests unitarios
- [ ] Función `anon_field(value)` implementada y con tests unitarios (incluyendo None, "", multi-token)
- [ ] Lista de columnas PII configurable al inicio del notebook `04_anonimizacion.ipynb`
- [ ] Los campos PII aparecen en el output con valores hasheados, **no eliminados**
- [ ] `anon_field(None)` → `None` (nulls se preservan)
- [ ] `anon_field("")` → `""` (strings vacíos se preservan)
- [ ] `anon_field("Juan Pérez")` → `"<hash1> <hash2>"` (multi-token funciona)
- [ ] `anon_field("12345678")` → `"<hash>"` (cédula = token único)
- [ ] El JOIN en la base anonimizada por cédula hasheada funciona entre tablas
- [ ] `out/comunidad_anonimizada.db` producido
- [ ] Test: `anon_token("pérez") == anon_token("perez")` (normalización de acentos)
- [ ] Test: `anon_token("JUAN") == anon_token("juan")` (case-insensitive)
- [ ] Script ejecutable desde CLI: `nbclick run notebooks/04_anonimizacion.ipynb`

---

## 6. Extensiones Futuras (no en scope v1)

- **Salt configurable via `.env`**: agregar `ANON_SALT` para resistencia a diccionario
- **Lista negra de tokens comunes**: no hashear artículos ("de", "la", "del") para mantener legibilidad parcial en direcciones
- **Longitud configurable del hash**: 8 chars → 32 bits; aumentar a 12 si el dataset crece significativamente

---

## 7. Impacto en el PRD

El criterio original de Fase 4:
> *"Campos PII (nombres, documentos, teléfonos, direcciones exactas) **eliminados** o hasheados"*

Se reemplaza por:
> *"Campos PII (nombres, documentos, teléfonos, direcciones exactas) **hasheados por token** usando `anon_field()`. Ningún campo es eliminado."*

El resto del PRD (Fase 5, Fase 6) no se ve afectado.
