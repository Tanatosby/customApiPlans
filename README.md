# Custom API CAPP – Árbol de Programas → Áreas → Reglas → Cursos

Este README describe **cómo construir por SQL** la jerarquía de CAPP (Banner) para exponerla vía una **Custom API** con el formato JSON solicitado. Está alineado al ejemplo de `documentacion_api_capp.json` y pensado para **todos los programas académicos que inician con `PR`** (ajustable).

> **Objetivo:** Por cada **Programa (SMAPRLE)** listar sus **Áreas (SMAPROG)**; por cada **Área (SMAAREA)** listar sus **Reglas**; y por cada **Regla** listar **Cursos** que satisfacen la(s) condición(es).

---

## 1) Alcance y Supuestos

- Se usa la **terminología de páginas** y **tablas** típica de CAPP (puede variar por versión):
  - **SMAPRLE** → tabla `SMRPRLE` (definición de Programa/Plan CAPP).
  - **SMAPROG** → tabla `SMRPAAP` (asignación de **Áreas** a un Programa).
  - **SMAAREA** → tablas `SMRACAA` (definición/uso de **Reglas** en un Área) y `SMBARUL` (catálogo/desc. de regla).
  - **Regla → Cursos** → tabla `SMRARUL` (líneas de regla que referencian cursos), `SCBCRSE` (título vigente del curso).
  - **Cursos migrados** → **fallback** a `SHRTCKN.SHRTCKN_CRSE_TITLE` cuando existe curso en Banner se convierte `SCBCRSE` aplicable.
- **Efectividad por término:** cuando corresponda, se filtra contra un término de catálogo (**`SOBCURR.SOBCURR_TERM_CODE_INIT`** en el ejemplo del usuario) o se usa el **máximo `EFF_TERM` ≤ término de catálogo**.
- Prefijo de programa **`PR%`** como ejemplo. Cambiar por lo que TI requiera.
- Los nombres de columnas pueden variar por release; validar en su instancia (señalamos columnas típicas).

---

## 2) Tablas y claves principales (resumen operativo)

| Lógica | Tabla (alias) | Campos clave (ejemplo) | Comentarios |
|---|---|---|---|
| Programa/Plan | `SMRPRLE p` | `p.SMRPRLE_PROGRAM`, `p.SMRPRLE_PROGRAM_DESC` | Página **SMAPRLE** |
| Catálogo (inicio) | `SOBCURR c` | `c.SOBCURR_PROGRAM`, `c.SOBCURR_TERM_CODE_INIT` | Catálogo de inicio por programa |
| Áreas por Programa | `SMRPAAP a` | `a.SMRPAAP_PROGRAM`, `a.SMRPAAP_AREA`, `a.SMRPAAP_AREA_PRIORITY` | Página **SMAPROG** |
| Reglas en Área | `SMRACAA r` | `r.SMRACAA_AREA`, `r.SMRACAA_RULE` | Página **SMAAREA** |
| Catálogo de Regla | `SMBARUL br` | `br.SMBARUL_RULE`, `br.SMBARUL_DESC` | Descripción legible de la regla |
| Cursos de Regla | `SMRARUL rc` | `rc.SMRARUL_RULE`, `rc.SMRARUL_SUBJ_CODE`, `rc.SMRARUL_CRSE_NUMB_LOW` (y/o HIGH) | Detalle de condiciones/cursos |
| Curso (título) | `SCBCRSE cr` | `cr.SCBCRSE_SUBJ_CODE`, `cr.SCBCRSE_CRSE_NUMB`, `cr.SCBCRSE_EFF_TERM`, `cr.SCBCRSE_TITLE` | Título vigente por término |
| Migrados (título) | `SHRTCKN ck` | `ck.SHRTCKN_SUBJ_CODE`, `ck.SHRTCKN_CRSE_NUMB`, `ck.SHRTCKN_TERM_CODE` | Título histórico/migrado |

> **Condiciones múltiples:** en `SMRARUL` suelen existir campos de **agrupación/secuencia** (por ej. `COND_SEQ`, `GROUP_NO`, `LINE_NO` según versión). Úsenlos para **separar** en el JSON **por condición** cuando una regla tiene varias. Valide los nombres exactos en su BD.

---

## 3) SQL por niveles

> **Nota:** Ajuste nombres de columnas que difieran en su release. Añada filtros por vigencia (`EFF_TERM`) y estado (activo) según sus normas.

### 3.1 Programas (SMRPRLE) + Término de Catálogo (SOBCURR)

```sql
-- Lista de Programas tipo PR% con descripción y término de catálogo inicial
SELECT
  p.SMRPRLE_PROGRAM                AS programa_academico,
  p.SMRPRLE_PROGRAM_DESC           AS nombre_programa,
  c.SOBCURR_TERM_CODE_INIT         AS periodo_de_catalogo
FROM SMRPRLE p
LEFT JOIN SOBCURR c
       ON c.SOBCURR_PROGRAM = p.SMRPRLE_PROGRAM
WHERE p.SMRPRLE_PROGRAM LIKE 'PR%'
ORDER BY p.SMRPRLE_PROGRAM;
```

### 3.2 Áreas por Programa (SMRPAAP)

```sql
-- Áreas y prioridad para un programa dado
SELECT
  a.SMRPAAP_PROGRAM  AS programa_academico,
  a.SMRPAAP_AREA     AS nombre_area,
  a.SMRPAAP_AREA_PRIORITY AS prioridad_area
FROM SMRPAAP a
WHERE a.SMRPAAP_PROGRAM = :programa   -- ej. 'PR02INME'
ORDER BY
  LPAD(a.SMRPAAP_AREA_PRIORITY, 3, '0'), a.SMRPAAP_AREA;
```

### 3.3 Reglas por Área (SMAAREA = SMRACAA + SMBARUL)

```sql
-- Reglas que el Área utiliza, con su descripción desde SMBARUL
SELECT
  r.SMRACAA_AREA     AS nombre_area,
  r.SMRACAA_RULE     AS nombre_regla,
  br.SMBARUL_DESC    AS descripcion_regla
FROM SMRACAA r
LEFT JOIN SMBARUL br
       ON br.SMBARUL_RULE = r.SMRACAA_RULE
WHERE r.SMRACAA_AREA = :area -- ej. 'PR02INME01'
ORDER BY r.SMRACAA_RULE;
```

### 3.4 Cursos por Regla (separando condiciones)

La granularidad por **condición** depende de su release. A menudo existen campos como:
- `SMRARUL_COND_SEQ` (secuencia de condición),
- `SMRARUL_GROUP_NO` (grupo lógico),
- `SMRARUL_SEQ_NO` / `LINE_NO` (orden dentro de la condición).

A continuación un patrón genérico con **COALESCE** para título (prioriza SCBCRSE y cae a SHRTCKN si no encuentra):

```sql
/* Cursos asociados a una Regla, agrupados por condición.
   Ajuste los nombres de columnas de agrupación a su versión (COND_SEQ/GROUP/LINE).
*/
WITH base AS (
  SELECT
    rc.SMRARUL_RULE                AS nombre_regla,
    rc.SMRARUL_SUBJ_CODE          AS subj_code,
    rc.SMRARUL_CRSE_NUMB_LOW      AS crse_numb_low,
    rc.SMRARUL_CRSE_NUMB_HIGH     AS crse_numb_high,
    -- Posibles identificadores de condición (ajuste a su release)
    rc.SMRARUL_COND_SEQ           AS cond_seq,
    rc.SMRARUL_GROUP_NO           AS group_no,
    rc.SMRARUL_SEQ_NO             AS line_no
  FROM SMRARUL rc
  WHERE rc.SMRARUL_RULE = :regla -- ej. 'MATE02006'
),
curso_titulo AS (
  SELECT
    b.*,
    /* Título desde SCBCRSE (vigente) */
    (SELECT cr.SCBCRSE_TITLE
       FROM SCBCRSE cr
      WHERE cr.SCBCRSE_SUBJ_CODE = b.subj_code
        AND cr.SCBCRSE_CRSE_NUMB = b.crse_numb_low
      ORDER BY cr.SCBCRSE_EFF_TERM DESC
      FETCH FIRST 1 ROWS ONLY) AS title_scbcrse,
    /* Fallback migrado desde SHRTCKN */
    (SELECT ck.SHRTCKN_CRSE_TITLE
       FROM SHRTCKN ck
      WHERE ck.SHRTCKN_SUBJ_CODE = b.subj_code
        AND ck.SHRTCKN_CRSE_NUMB = b.crse_numb_low
      ORDER BY ck.SHRTCKN_TERM_CODE DESC
      FETCH FIRST 1 ROWS ONLY) AS title_migrado
  FROM base b
)
SELECT
  nombre_regla,
  cond_seq,
  group_no,
  line_no,
  subj_code || LPAD(crse_numb_low, 5, '0')     AS nombre_curso,
  COALESCE(title_scbcrse, title_migrado, 'SIN TÍTULO') AS descripcion_del_curso
FROM curso_titulo
ORDER BY cond_seq NULLS FIRST, group_no NULLS FIRST, line_no NULLS FIRST;
```

> **Buscar condicionales en columnas como `COND_SEQ`/`GROUP_NO`/`SEQ_NO`:** o podemos identificar los campos equivalentes en `SMRARUL` (p. ej. `SMRARUL_REQ_GROUP`, `SMRARUL_REQ_SEQ`), y reemplázarlos en el CTE `base` y en el `ORDER BY`.

---

## 4) Composición del JSON (desde SQL)

Con las consultas anteriores, la API puede **ensamblar** el árbol. Dos aproximaciones comunes:

1. **Consulta por niveles + ensamblado en código** (recomendado para claridad y cache):  
   - Query Programas → por cada programa, query Áreas → por cada área, query Reglas → por cada regla, query Cursos (con condición).
2. **Consulta única con JOINs** y luego **agrupar** en backend para formar el JSON (útil si se paginará por profundidad).

### Ejemplo de JOIN total (para backend que agrega)

```sql
SELECT
  p.SMRPRLE_PROGRAM                      AS programa_academico,
  p.SMRPRLE_PROGRAM_DESC                 AS nombre_programa,
  c.SOBCURR_TERM_CODE_INIT               AS periodo_de_catalogo,
  a.SMRPAAP_AREA                         AS nombre_area,
  a.SMRPAAP_AREA_PRIORITY                AS prioridad_area,
  r.SMRACAA_RULE                         AS nombre_regla,
  br.SMBARUL_DESC                        AS descripcion_regla,
  rc.SMRARUL_COND_SEQ                    AS cond_seq,
  rc.SMRARUL_GROUP_NO                    AS group_no,
  rc.SMRARUL_SEQ_NO                      AS line_no,
  rc.SMRARUL_SUBJ_CODE                   AS subj_code,
  rc.SMRARUL_CRSE_NUMB_LOW               AS crse_numb_low
FROM SMRPRLE p
LEFT JOIN SOBCURR c
       ON c.SOBCURR_PROGRAM = p.SMRPRLE_PROGRAM
LEFT JOIN SMRPAAP a
       ON a.SMRPAAP_PROGRAM = p.SMRPRLE_PROGRAM
LEFT JOIN SMRACAA r
       ON r.SMRACAA_AREA = a.SMRPAAP_AREA
LEFT JOIN SMBARUL br
       ON br.SMBARUL_RULE = r.SMRACAA_RULE
LEFT JOIN SMRARUL rc
       ON rc.SMRARUL_RULE = r.SMRACAA_RULE
WHERE p.SMRPRLE_PROGRAM LIKE 'PR%'
ORDER BY p.SMRPRLE_PROGRAM,
         LPAD(a.SMRPAAP_AREA_PRIORITY, 3, '0'),
         a.SMRPAAP_AREA,
         r.SMRACAA_RULE,
         rc.SMRARUL_COND_SEQ,
         rc.SMRARUL_GROUP_NO,
         rc.SMRARUL_SEQ_NO;
```

> El **backend** completará `descripcion_del_curso` juntando contra `SCBCRSE`/`SHRTCKN` (o con subconsultas como en 3.4).

---

## 5) Diseño de la Custom API

### 5.1 Endpoints sugeridos

- **(A) Árbol completo de un programa**  
  `GET /capp/tree?program=PR02INME&catalogTerm=000000`  
  Devuelve el JSON del ejemplo con **Áreas → Reglas → Cursos**, separando **condiciones** dentro de cada regla.  
  Parámetros opcionales: `includeMigratedTitles=true|false`, `maxDepth=areas|rules|courses`.

- **(B) Listado de programas**  
  `GET /capp/programs?prefix=PR&active=true`

- **(C) Áreas de un programa**  
  `GET /capp/programs/{program}/areas`

- **(D) Reglas de un área**  
  `GET /capp/areas/{area}/rules`

- **(E) Cursos de una regla**  
  `GET /capp/rules/{rule}/courses?byCondition=true`

### 5.2 Respuesta (ejemplo resumido)

```json
{
  "programa_academico": "PR02INME",
  "nombre_programa": "ING. MECÁNICO-ELÉCTRICA",
  "periodo_de_catalogo": "000000",
  "areas": [
    {
      "nombre_area": "PR02INME01",
      "prioridad_area": "30",
      "reglas_area": [
        {
          "nombre_regla": "MATE02006",
          "descripcion_regla": "CÁLCULO ELEMENTAL",
          "condiciones": [
            {
              "cond_seq": 1,
              "cursos": [
                {
                  "nombre_curso": "MATE02006",
                  "descripcion_del_curso": "CÁLCULO ELEMENTAL"
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

> Si no desean anidar por condición: flatten en la regla (pero incluir `cond_seq` en cada curso).

---

## 6) Consideraciones técnicas

- **Vigencias (EFF_TERM):** para títulos y planes vigentes, usar **máximo `EFF_TERM` ≤ `catalogTerm`**. Si no hay `catalogTerm`, usar el **actual**.
- **Cursos migrados:** cuando falte `SCBCRSE`, usar `SHRTCKN_CRSE_TITLE` (fallback). El ejemplo ya contempla ambos.
- **Prioridad del Área:** respetar `SMRPAAP_AREA_PRIORITY` para ordenar en el JSON.
- **Rendimiento:** agregar índices si es necesario sobre `SMRPAAP(PROGRAM)`, `SMRACAA(AREA)`, `SMRARUL(RULE)`, y fechas de efectividad.
- **Seguridad:** exponer solo lectura; registrar auditoría (quién, cuándo, qué programa/área consultó).
- **Paginación:** permitir `maxDepth` y `pageSize` si el árbol es grande; o filtrar por `program` específico.
- **Multicampus/Planes:** si manejan múltiples variantes por campus/periodo, incluyan filtros adicionales (`camp_code`, `term_code`).

---

## 7) Checklist para TI (para construir el API)

1. Validar nombres exactos de columnas de **condición** en `SMRARUL` (p. ej. `COND_SEQ`, `GROUP_NO`, `SEQ_NO`).  
2. Confirmar si `SOBCURR` es la tabla correcta del **catálogo** en su release; si no, usar `SOACURR/SORLCUR` equivalente.  
3. Implementar **joins de títulos** con el patrón `COALESCE(SCBCRSE_TITLE, SHRTCKN_CRSE_TITLE)`.  
4. Respetar **orden**: Programa → Prioridad de Área → Regla → Condición → Línea.  
5. Implementar endpoints (A–E) y **tests** con los ejemplos de `documentacion_api_capp.json`.  
6. Loggear consultas y parámetros (program, term, depth) para soporte.

---

## 8) Preguntas abiertas (para acordar con TI)

- ¿Cada elemento llevará **ID interno** además del código (program/area/rule)? (Recomendable para claves estables del API).
- ¿Cuál será el **término de catálogo** por defecto si no se envía (hoy, último, o específico por programa)?
- ¿Se **expone condición** como nivel explícito en el JSON (recomendado) o se **aplana**?
- ¿Qué hacer con **créditos/electivas** cuando la regla no referencia cursos (p. ej. por atributo o nivel)? ¿Se incluye un nodo de **criterios**?

---

## 9) Apéndice – Snippet para título del curso por vigencia

```sql
SELECT cr.SCBCRSE_TITLE
FROM SCBCRSE cr
WHERE cr.SCBCRSE_SUBJ_CODE = :subj
  AND cr.SCBCRSE_CRSE_NUMB = :crse
  AND cr.SCBCRSE_EFF_TERM <= :catalogTerm
ORDER BY cr.SCBCRSE_EFF_TERM DESC
FETCH FIRST 1 ROWS ONLY;
```

**Fallback migrado:**

```sql
SELECT ck.SHRTCKN_CRSE_TITLE
FROM SHRTCKN ck
WHERE ck.SHRTCKN_SUBJ_CODE = :subj
  AND ck.SHRTCKN_CRSE_NUMB = :crse
ORDER BY ck.SHRTCKN_TERM_CODE DESC
FETCH FIRST 1 ROWS ONLY;
```

---

**Autor:** Equipo Registros Académicos – UDEP  
**Contacto TI:** (completar)
# customApiPlans
Para poder obtener un Custom API que tenga el detalle de los planes de estudios. 
