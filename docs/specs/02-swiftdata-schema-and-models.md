# Spec 02 — SwiftData schema and models

## Objetivo

Definir el esquema SwiftData de `Alumno`, catálogos fijos (sede, cadencia), importes por defecto y propiedades calculadas de deuda/estado, de modo que dashboard, alta y CSV usen el mismo contrato.

## Contexto y dependencias

- **Fuente de verdad:** `docs/prds/PRD_Control_Escolar_Nova_Impulsa.md` (conceptos de dominio, RF-007 a RF-012, RNF-009).
- **Debe existir:** spec 01 (proyecto, carpetas `Models/` `Utilities/`, `NovaImpulsaApp` con `ModelContainer`).
- **Un operador, un iPhone, matrícula única en el dispositivo.** No hay entidad Usuario ni Sede persistida.
- Costo default por alumno: **$11,549** = $700 + $250 + 3×$1,200 + $6,999.

Si spec 01 dejó `PlaceholderModel`, elimínalo y registra `Alumno`.

## Alcance

### In scope

- Modelo `@Model Alumno` con todos los campos persistidos del PRD.
- Enums/catálogos: `CadenciaCobro`, `EstadoPago`, `SedeCatalog`.
- Constantes de importes default.
- Propiedades calculadas: total esperado, pagado, saldo, certificación pendiente, vencido, estado, fecha efectiva de próximo pago.
- `#Unique` en `matricula` (o equivalente documentado si el toolchain no lo soporta).
- `ModelContainer` apuntando a `Alumno.self`.
- Helpers de formato `es_MX` para moneda y fecha.
- Unit tests de cálculos con datos en memoria (sin UI).

### Out of scope

- Pantallas, formularios, CSV, WhatsApp, notificaciones.
- Modelo `Pago` separado (los 6 conceptos van **aplanados** en `Alumno`, alineado al CSV del PRD).
- Catálogo administrable de sedes/asesores.
- Baja lógica / `isActivo`.

## Tareas

1. Crear `Utilities/AppConstants.swift`:
   - `importeMatriculaDefault: Decimal = 700`
   - `importeGuiaDefault: Decimal = 250`
   - `importeColegiaturaDefault: Decimal = 1200`
   - `importeCertificacionDefault: Decimal = 6999`
   - `totalDefault: Decimal = 11549`
   - `horaAlarmaDefault: (hour: 9, minute: 0)`
   - `semanasCertificacion = 12`
2. Crear `Models/SedeCatalog.swift`: lista fija (cambiar valores no cambia el modelo):
   - `["Sede Centro", "Sede Norte", "Sede Sur"]`
   - `static let defaultSede = all[0]`
3. Crear `Models/CadenciaCobro.swift`: `semanal`, `quincenal`, `mensual` (raw `String`, `CaseIterable`). Método `avanzar(_ date: Date, calendar: Calendar) -> Date`:
   - semanal: +7 días
   - quincenal: +15 días
   - mensual: +1 mes calendario (`calendar.date(byAdding: .month, value: 1, to:)`)
4. Crear `Models/EstadoPago.swift`: `alDia`, `vencido`, `liquidado` con `label` en español: «Al día», «Vencido», «Liquidado».
5. Crear `Models/Alumno.swift` `@Model`:

   Persistidos:

   | Campo | Tipo | Default / regla |
   |-------|------|-----------------|
   | `id` | `UUID` | `UUID()` |
   | `nombreCompleto` | `String` | |
   | `telefonoContacto` | `String` | |
   | `telefonoAlterno` | `String` | `""` |
   | `matricula` | `String` | **única** |
   | `sede` | `String` | valor de `SedeCatalog` |
   | `asesor` | `String` | `""` |
   | `fechaInscripcion` | `Date` | |
   | `cadencia` | `CadenciaCobro` | |
   | `fechaProximoPago` | `Date` | |
   | `horaAlarma` | `DateComponents` persistidos como `alarmaHour: Int = 9`, `alarmaMinute: Int = 0` | |
   | `matriculaPagada` | `Bool` | `false` |
   | `guiaPagada` | `Bool` | `false` |
   | `col1Pagada` | `Bool` | `false` |
   | `col2Pagada` | `Bool` | `false` |
   | `col3Pagada` | `Bool` | `false` |
   | `importeMatricula` | `Decimal` | 700 |
   | `importeGuia` | `Decimal` | 250 |
   | `importeCol1` | `Decimal` | 1200 |
   | `importeCol2` | `Decimal` | 1200 |
   | `importeCol3` | `Decimal` | 1200 |
   | `importeCertificacion` | `Decimal` | 6999 |
   | `certificacionAbonado` | `Decimal` | 0 |
   | `fechaCertificacion` | `Date` | inscripción + 12 semanas |

   SwiftData: usar `#Unique<Alumno>([\Alumno.matricula])` en iOS 17.4+ si el proyecto lo permite; si no, documentar en comentario y validar unicidad en ViewModels (spec 05). Preferir `Decimal` nativo; si SwiftData del SDK falla con `Decimal`, persistir `Int` en centavos (`70000` = $700.00) y exponer `Decimal` en computed. **Elegir una y usarla en toda la app.** Recomendado: **centavos `Int`** para evitar bugs de SwiftData + Decimal.

6. Implementar calculados (en `Alumno` o en `Utilities/PaymentMath.swift` llamados por `Alumno`; spec 06 podrá extraerlos, pero **las fórmulas deben existir ya y ser testeables**):

   - `totalEsperado` = suma de los 6 importes.
   - `totalPagado` = (matriculaPagada ? importeMatricula : 0) + igual para guía y cols + `min(certificacionAbonado, importeCertificacion)`.
   - `saldo` = `totalEsperado - totalPagado` (≥ 0).
   - `certificacionPendiente` = `max(importeCertificacion - certificacionAbonado, 0)`.
   - `fechaVencimientoMatricula` / `Guia` = `fechaInscripcion`.
   - `fechaVencimientoCol(n)` = inscripción + n meses (n = 1, 2, 3).
   - `estaVencido(en now: Date = .now)`:
     - matrícula/guía no pagadas y `startOfDay(inscripción) < startOfDay(now)`
     - col N no pagada y `startOfDay(inscripción+N meses) < startOfDay(now)`
     - `certificacionPendiente > 0` y `startOfDay(fechaCertificacion) < startOfDay(now)`
   - `estadoPago(en now:)`: si `saldo == 0` → `liquidado`; si `estaVencido` → `vencido`; si `saldo > 0` y no vencido → `alDia`.
   - `fechaProximoPagoEfectiva`: `fechaProximoPago` si está set (siempre lo estará en alta); fallback: mínima fecha de vencimiento entre cargos no cubiertos.

7. Crear `Utilities/Formatters.swift`:
   - moneda: locale `es_MX`, `$` + 2 decimales (p. ej. `$1,200.00`).
   - fecha: `dd/MM/yyyy`.
   - Usar `Calendar.current` y zona del dispositivo.

8. Factory `Alumno.makeDefault(...)` para tests/previews: defaults de importes, `fechaCertificacion = inscripción + 12 weeks`, checkboxes false, abono 0.

9. En `NovaImpulsaApp`: `.modelContainer(for: Alumno.self)`. Sin CloudKit. En Previews: `ModelConfiguration(isStoredInMemoryOnly: true)`.

10. Tests en `NovaImpulsaTests/AlumnoMathTests.swift` (tabla del PRD):
    - Alta sin pagos: esperado 11549, pagado 0, saldo 11549, no liquidado.
    - Matrícula + Guía pagadas: pagado 950, saldo 10599.
    - Abono certificación 2000: certificación pendiente 4999, saldo baja 2000.
    - Abono 8000 con tope 6999: pagado de cert = 6999 (el clamp puede vivir en spec 06; **si el modelo no clampa al asignar**, el computed `totalPagado` sí debe usar `min`).
    - Inscripción `2026-03-01` → `fechaCertificacion` default `2026-05-24`.
    - Col1 no pagada, inscripción hace > 1 mes, `now` actual → `estaVencido == true`, estado `vencido`.
    - Saldo 0 → `liquidado`, no vencido para KPIs de “al día”.

## Criterios de aceptación

- Compila con `Alumno` como único modelo del container.
- Un test crea dos `Alumno` en store in-memory; los calculados coinciden con los montos del PRD (± $0.00).
- `SedeCatalog.all` tiene ≥ 1 valor; no es texto libre en el modelo (el campo es `String` pero el catálogo es la fuente de opciones).
- No hay `Pago` `@Model`.
- Dinero no usa `Double` en la capa persistida (Decimal o centavos `Int`).
- Fechas de colegiatura usan meses calendario, no 30 días fijos.
- `PlaceholderModel` de spec 01 ya no existe.

## Notas técnicas

- Comparar vencimientos con **inicio del día** local para no marcar vencido el mismo día a las 00:01 por timezone.
- `Calendar.current.date(byAdding: .weekOfYear, value: 12, to: inscripción)` para certificación; el caso 01/03/2026 → 24/05/2026 debe pasar el test (12×7 = 84 días).
- KPI “alumnos al día” = `saldo > 0 && !estaVencido` (no incluye liquidados).
- KPI “pagos pendientes” = `saldo > 0`.
- KPI “vencidos” = `estaVencido`.
- KPI “certificación pendiente” agregado = suma de `certificacionPendiente`.
- No implementar UI. Los ViewModels de specs 03–07 solo leerán estas propiedades.
