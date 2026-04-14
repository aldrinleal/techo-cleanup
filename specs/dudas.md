# Dudas y Preguntas Abiertas

Preguntas que bloquean o podrían afectar decisiones de diseño. Resolver antes de implementar las fases correspondientes.

---

## D-01 — Rol de Kobotoolbox en el flujo actual

**Estado**: Abierta
**Bloquea**: Fase 1 (ingesta), modelo de datos final

El equipo menciona Kobotoolbox en la discusión, pero la pestaña "Caracterizaciones" viene de un Google Form y las 3 pestañas de comunidad son seguimientos manuales.

- ¿Se usa Kobotoolbox actualmente en algún paso del flujo?
- ¿Es una herramienta futura (target de migración) o ya está en producción?
- Si está activo, ¿a qué Google Sheet alimenta y qué columnas produce?

---

## D-02 — Esquema del Google Form de Caracterizaciones

**Estado**: Abierta
**Bloquea**: Fase 2 (modelado), definición de PII fields

La pestaña "Caracterizaciones" es una exportación del Google Form de encuesta inicial.

- ¿Hay acceso al formulario de Google Forms para ver el esquema de preguntas?
- Conocer el esquema permite mapear columnas → campos antes de ver el ODS, y acelera la identificación de PII.

---

## D-03 — Diferencia de columnas entre las 3 pestañas de comunidad

**Estado**: Abierta
**Bloquea**: Fase 1 (ingesta), Fase 2 (modelado de `ENTREVISTA_SEGUIMIENTO`)

Las pestañas Vereda El Granizal, Manrique La Honda y Vereda La Nueva Jerusalen son entrevistas de seguimiento, pero probablemente tienen columnas distintas entre sí o distintas a "Caracterizaciones".

- ¿Las 3 pestañas tienen el mismo esquema o cada comunidad tiene su propio formato?
- ¿Hay columnas en común con "Caracterizaciones" que permitan hacer JOIN por familia?
- ¿Cuál es el campo que identifica a una familia de forma única (¿cédula?, ¿nombre + dirección?)

---
