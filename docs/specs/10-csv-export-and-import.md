# Spec 10 — CSV export and import

## Objetivo

Importar y exportar el padrón en CSV UTF-8 compatible con Excel, con las columnas del PRD, sin overwrite de alumnos existentes, y con resumen de filas OK/error.

## Contexto y dependencias

- **Fuente de verdad:** PRD RF-026 a RF-029 y criterios CSV.
- **Debe existir:** spec 03 (botones Importar / Exportar / Excel no-op), spec 02/06 (`Alumno` + calculator). Tras importar, KPIs y lista deben moverse solos (`@Query`).
- **Exportar y Excel generan el mismo archivo.** No hay `.xlsx`.
- Importar **solo altas nuevas**. Matrícula duplicada = error de fila, no update.
- Importar no borra ni modifica existentes (RF-029).
- Backup de negocio = este CSV (no iCloud de la app).

Columnas mínimas (orden de export, header exacto):

```
matricula,nombre,telefono,telefono_alterno,sede,asesor,fecha_inscripcion,cadencia,fecha_proximo_pago,matricula_pagada,guia_pagada,col1_pagada,col2_pagada,col3_pagada,certificacion_abonado,certificacion_importe,certificacion_fecha,total_esperado,total_pagado,saldo,estado
```

- Booleanos: `si` / `no` (minúsculas).
- Fechas: `dd/MM/yyyy`.
- Cadencia: `semanal` | `quincenal` | `mensual`.
- Estado exportado: `Al día` | `Vencido` | `Liquidado` (label de `EstadoPago`).
- Decimales: punto o el formatter que Excel en MX acepte; **exportar con punto** y 2 decimales (`11549.00`) para parseo estable. Documentar en comentario del CSV (no hace falta fila extra).
- `total_esperado`, `total_pagado`, `saldo`, `estado` en **export** son informativos. En **import** se omiten o se ignoran y se **recalculan**.

Import — campos obligatorios de fila: `matricula`, `nombre`, `telefono`, `sede`, `cadencia`, `fecha_inscripcion`, `fecha_proximo_pago`.

Defaults si faltan opcionales: teléfonos alterno vacío, asesor vacío, pagadas `no`, abono `0`, importe cert `6999`, fecha cert = inscripción + 12 semanas si falta.

Validación de fila: matrícula no vacía y no existente **ni duplicada dentro del mismo archivo**; sede debe estar en `SedeCatalog.all`; cadencia válida; fechas parseables; abono ≥ 0.

Resumen post-import: alert «Importadas: N. Errores: M» + lista corta de matrícula + motivo (máx. ~10 líneas + «…»).

## Alcance

### In scope

- `Services/CSVExporter.swift`, `Services/CSVImporter.swift`.
- Share sheet (`UIActivityViewController` / `ShareLink`) para Exportar y Excel.
- `fileImporter` / `UIDocumentPicker` para Importar (`comma-separated-text`, `.csv`, `public.text`).
- Cablear los 3 botones del dashboard.
- Tests con strings CSV (sin Excel).

### Out of scope

- Generar `.xlsx` (CoreXLSX, etc.).
- Merge/overwrite.
- Importar fotos o múltiples hojas.
- Sync en segundo plano.

## Tareas

1. `CSVExporter.export(alumnos:now:) -> String` con header + filas. Escapar comillas (`"` → `""`) y envolver campos con coma.
2. Escribir archivo temporal `NovaImpulsa_Alumnos_yyyyMMdd.csv` en `FileManager.temporaryDirectory` y compartir (`ShareLink` iOS 16+ o wrapper UIKit).
3. Botones **Exportar** y **Excel**: misma función. Excel no cambia el mime; el usuario abre el CSV con Excel.
4. `CSVImporter.parse(csv:existingMatriculas:) -> (ok: [AlumnoDraft], errors: [CSVRowError])`.
5. `AlumnoDraft` → `Alumno` insert. Recalcular con `PaymentCalculator`. Clamp abono.
6. Transacción: insertar todos los OK; no insertar las filas error. Un error no aborta el archivo entero.
7. Duplicado intra-archivo: primera fila gana; siguientes con misma matrícula = error.
8. Tests:
   - Export roundtrip: 1 alumno conocido → CSV contiene header y matrícula.
   - Import 3 filas nuevas válidas → 3 `Alumno` y KPIs: pendientes aumentan acorde.
   - 2 nuevas + 1 matrícula ya en store → +2, error menciona la duplicada.
   - Reimportar el export de un alumno existente → 0 inserts, alumno original mismo saldo.
   - Totales en CSV ignorados: fila con `saldo` mentiroso se recalcula.
   - Campo con coma en nombre (`García, Ana`) roundtrip.
9. Empty export: CSV solo header; share sigue permitido.
10. UTF-8 con BOM (`\u{FEFF}`) al inicio del archivo de export para que Excel Windows abra acentos bien.

## Criterios de aceptación

- Exportar y Excel producen el mismo CSV y abren la hoja de compartir.
- Header coincide carácter a carácter con la lista de RF-026 (nombres de columna del PRD).
- Import de 3 alumnos nuevos los muestra en lista y mueve KPIs.
- Import mixto (1 duplicada, 2 nuevas) crea 2 y resume la matrícula rechazada.
- Reimportar no cambia saldo ni checkboxes del existente.
- Archivo inválido (sin header) → 0 inserts + error global, sin crash.
- Offline: todo local, sin red.

## Notas técnicas

- Parser: no usar `split(",")` ingenuo; respetar comillas. Implementar un parser mínimo o `enumerateLines` + state machine. Prohibido crash por fila mal formada.
- Locale: parsear Decimal con `.` ; si un campo trae `,` decimal y no está quoted como miles, intentar fallback.
- `fileImporter` requiere que el CSV esté en Files / iCloud Drive del usuario; en simulador arrastrar el archivo.
- No usar encabezados localizados distintos al PRD (rompe el import).
- Tras insert masivo, `try modelContext.save()`.
- Permiso de notificaciones: import **no** está obligado a programar alarmas para cada fila (evitar 200 diálogos). Spec 08 cubre save UI. Opcional: tras import, loop `schedule` silencioso si el permiso ya está granted; si se implementa, no pedir permiso en medio del import. Recomendado: **sí** schedule silencioso si `authorizationStatus == .authorized`, para no dejar huérfanos. Si denied, no avisar N veces; un solo texto en el resumen «Recordatorios no programados (sin permiso)».
