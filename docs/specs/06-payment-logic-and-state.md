# Spec 06 — Payment logic and state

## Objetivo

Dejar un único módulo de dominio testeable para montos, vencimientos, estado, clamp de abonos y fechas derivadas, y usarlo desde modelos y formularios para que KPIs y filas coincidan siempre con el PRD.

## Contexto y dependencias

- **Fuente de verdad:** PRD estructura de costos, cálculo de deuda, RF-007 a RF-012, criterios de alta/certificación/vencimiento.
- **Debe existir:** spec 02 (campos de `Alumno`) y spec 05 (formulario que hoy puede tener la lógica inline).
- Esta spec **no añade pantallas**. Extrae y endurece reglas. Si spec 05 duplica fórmulas, reemplazarlas por llamadas a este módulo.

Reglas canónicas (copiar a tests):

```
totalEsperado = Σ importes (6 conceptos)
totalPagado   = Σ importes de binarios pagados + min(abonoCert, importeCert)
saldo         = max(totalEsperado - totalPagado, 0)
certPendiente = max(importeCert - abonoCert, 0)

vencido si alguno:
  !matriculaPagada && day(inscripcion) < day(now)
  !guiaPagada      && day(inscripcion) < day(now)
  !colNPagada      && day(inscripcion + N months) < day(now)   // N=1,2,3
  certPendiente>0  && day(fechaCertificacion) < day(now)

estado: saldo==0 → liquidado
        vencido   → vencido
        else      → alDia   // implica saldo>0

proximoPago propuesto: inscripcion + (7d | 15d | 1 mes)
fechaCert default:     inscripcion + 12 weeks
```

Binarios: marcar checkbox = pagado al 100% de su importe. Desmarcar = no pagado, recalcular. No hay abonos a colegiatura.

Cadencia **no** crea 12 cargos ni parte los $1,200.

## Alcance

### In scope

- `Utilities/PaymentCalculator.swift` (struct/enum sin SwiftUI).
- Clamp `clampAbono(_ proposed, tope:)`.
- Helpers de fechas de vencimiento y propuesta de próximo pago / certificación.
- Aplicar “marcar / desmarcar” y “registrar abono” sobre un `Alumno` (mutators).
- Reemplazar lógica duplicada en `Alumno` computed y `AddStudentViewModel`.
- XCTest exhaustivos (tabla del PRD).

### Out of scope

- UI nueva, CSV, WhatsApp, notificaciones.
- Recargos, descuentos, prorrateo, intereses.
- Historial de abonos (un solo acumulado `certificacionAbonado` es suficiente).
- Editar importes en UI (si no existe, mutators sí deben aceptar nuevo tope para cuando spec 07 lo exponga).

## Tareas

1. Crear `PaymentCalculator` con métodos estáticos puros (`Decimal`, `Date`, `Calendar`, `now` inyectable).
2. `func applyAbono(current: Decimal, addition: Decimal, cap: Decimal) -> Decimal` → nunca > cap, nunca < 0.
3. `func setBinaryPaid(_ key: BinaryConcept, paid: Bool)` sobre alumno.
4. `func recomputeNoop` — documentar que los computed de `Alumno` delegan 100% en `PaymentCalculator`.
5. Refactor: `Alumno.totalEsperado` etc. llaman a `PaymentCalculator`.
6. Refactor: `AddStudentViewModel.proposeProximoPago` / certificación / clamp llaman al calculator.
7. Tests `PaymentCalculatorTests` (fechas ancladas, no `Date.now` flaky salvo tests de vencido con `now` fijo):

   | Caso | Expect |
   |------|--------|
   | Todo default, nada pagado | esperado 11549, pagado 0, saldo 11549, estado alDía si inscripción=now, certPend 6999 |
   | Matrícula+Guía pagadas | pagado 950, saldo 10599 |
   | Abono 2000 | certPend 4999, saldo −2000 vs sin abono |
   | Abono 2000 + otro 6000 | abonado=6999, certPend 0 |
   | Abono −10 | se ignora o queda 0 |
   | Desmarcar Matrícula después de pagada | pagado baja 700 |
   | inscripción 2026-03-01 | cert 2026-05-24 |
   | cadencia semanal, misma inscripción, sin override | próximo 2026-03-08 |
   | cadencia quincenal | 2026-03-16 |
   | cadencia mensual | 2026-04-01 |
   | col1 impaga, inscripción 2026-01-01, now 2026-03-01 | vencido |
   | todo pagado (5 checks + abono 6999) | liquidado, saldo 0, no vencido |
   | liquidado no cuenta como al día | estado liquidado |

8. Añadir test de integración ligero: insertar vía ViewModel de alta y leer `alumno.saldo` == calculator.

## Criterios de aceptación

- No queda una segunda copia de las fórmulas de saldo/estado en Views.
- Todos los casos de la tabla pasan.
- Desmarcar un checkbox en memoria (mutator) cambia `totalPagado` inmediatamente.
- Abono nunca persiste por encima del tope si se usa `applyAbono`.
- Dashboard y fila, con los mismos alumnos, muestran el mismo saldo (misma función).
- Compila; tests de spec 02 que se rompan por el refactor se actualizan y siguen verdes.

## Notas técnicas

- `startOfDay` con `Calendar.current`. Tests deben fijar `TimeZone` (p. ej. `America/Mexico_City`) o usar `calendar` inyectado para el caso 01/03/2026.
- Colegiaturas: `date(byAdding: .month, value: n, to: startOfDay(inscripcion))`.
- `Decimal` / centavos: el calculator debe hablar el **mismo** tipo que spec 02.
- No usar `Double` intermedio (`0.1` errors).
- El calculator no importa SwiftData; recibe structs o campos sueltos para poder testear sin container. Un overload que acepta `Alumno` puede vivir en un archivo `PaymentCalculator+Alumno.swift` en `Utilities/`.
