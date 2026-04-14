# Dudas y Preguntas Abiertas

Preguntas que bloquean o podrían afectar decisiones de diseño. Resolver antes de implementar las fases correspondientes.

---

## D-01 — Rol de Kobotoolbox en el flujo actual

**Estado**: ✅ Resuelta (Daniel González, 2026-04-13)
**Bloquea**: ~~Fase 1~~

Kobo SÍ está activo en el flujo actual. Es la herramienta de campo que se usa **después** del Google Form inicial:
1. Google Form → info básica → Sheet 1 (Caracterizaciones)
2. Voluntarios van al barrio → encuesta **Kobo** → info adicional → Sheet 2 (comunidad)
3. Kobo data determina qué familias se seleccionan para asignación
4. Sheet 2 se actualiza con las familias asignadas

Pendiente aún (ver D-02): qué columnas específicas produce Kobo en Sheet 2.

---

## D-02 — Esquema del Google Form de Caracterizaciones

**Estado**: Abierta
**Bloquea**: Fase 2 (modelado), definición de PII fields

La pestaña "Caracterizaciones" es una exportación del Google Form de encuesta inicial.

- ¿Hay acceso al formulario de Google Forms para ver el esquema de preguntas?
- Conocer el esquema permite mapear columnas → campos antes de ver el ODS, y acelera la identificación de PII.

---

## D-03 — Identificador común entre pestañas (clave de JOIN)

**Estado**: Abierta — CRÍTICA
**Bloquea**: Fase 1 (ingesta), Fase 2 (modelado), Fase 3 (deduplicación)

El proceso real tiene 2 hojas que deben conectarse (Sheet 1 de Google Form + Sheet 2 de Kobo). Sin un identificador común, `caracterizaciones` y `comunidades` son tablas desconectadas y el pipeline pierde su valor.

- ¿Cuál es el campo que identifica a una familia de forma única en ambas hojas? (¿cédula?, ¿nombre completo?, ¿un ID generado por el Form?)
- ¿Las 3 pestañas de comunidad tienen el mismo esquema entre sí, o cada comunidad tiene su propio formato de columnas?
- ¿Hay familias en las pestañas de comunidad (Kobo) que NO están en Caracterizaciones? (familias que hicieron la encuesta de campo sin haber llenado el Form inicial)

> La Fase 1 reportará cuántas filas de comunidades no tienen match en caracterizaciones, lo que ayudará a responder la última pregunta empíricamente.

---
