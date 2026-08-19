# Spec 05 — Add student form UI

## Objetivo

Implementar el formulario de nuevo alumno, validaciones, plan de pagos inicial (checkboxes + abono), persistencia SwiftData y retorno al dashboard con el alumno visible.

## Contexto y dependencias

- **Fuente de verdad:** PRD RF-001, RF-002, RF-003, RF-007, RF-008, RF-009, RF-010, RF-019.
- **Debe existir:** spec 02 (`Alumno`, catálogos, factory, unicidad de matrícula) y spec 03/04 (botón Nuevo alumno + lista).
- Spec 06 extraerá `PaymentCalculator` y tests de borde; **esta spec ya debe guardar datos correctos** para no dejar altas incompletas.
- Specs 08/09 aún no: al guardar **no** programes notificaciones ni abras WhatsApp.

Campos del formulario:

| Campo | UI | Obligatorio | Default |
|-------|-----|-------------|---------|
| Nombre completo | `TextField` | sí | vacío |
| Teléfono contacto | `TextField` `.phonePad` | sí | vacío |
| Teléfono alterno | `TextField` `.phonePad` | no | vacío |
| Matrícula | `TextField` | sí, única | vacío |
| Sede | `Picker` `SedeCatalog.all` | sí | primera sede |
| Asesor | `TextField` | no | vacío |
| Fecha inscripción | `DatePicker` (solo fecha) | sí | hoy |
| Cadencia | `Picker` semanal / quincenal / mensual | sí | mensual |
| Fecha próximo pago | `DatePicker` | sí | inscripción + intervalo de cadencia |
| Checkboxes | Matrícula, Guía, Colegiatura 1, 2, 3 | no | off |
| Abono certificación | `TextField` decimal | no | 0 |
| Fecha certificación | `DatePicker` | sí | inscripción + 12 semanas |

Importes default al insertar: 700, 250, 1200, 1200, 1200, 6999. En esta spec los importes **no necesitan editor de monto** (el PRD permite editarlos; si no hay UI de importe, persistir defaults; spec 07 puede añadir edición de importes si cabe en el mismo formulario — **mínimo de esta spec: defaults**).

Cadencia → propuesta de `fechaProximoPago` (RF-003):

- Al cambiar cadencia **o** fecha de inscripción, recálculo: semanal +7d, quincenal +15d, mensual +1 mes.
- Si el usuario ya editó a mano `fechaProximoPago`, no pisar en cada tecla del nombre. Regla: pisar solo cuando cambian cadencia o inscripción, o en el primer `onAppear`. Documentar `userOverrideProximoPago: Bool` si hace falta.

Abono: no permitir negativo. Si el usuario escribe más que 6999, recortar al tope al guardar (spec 06 formaliza el clamp; hazlo ya para no persistir basura).

## Alcance

### In scope

- `AddStudentView` + `AddStudentViewModel`.
- Navegación desde el `+` del dashboard (reemplazar el placeholder de spec 03).
- Validación visual por campo.
- `modelContext.insert` + `save`.
- Dismiss al guardar OK.

### Out of scope

- Editar alumno existente (spec 07).
- Notificación local al guardar (spec 08).
- Import CSV (spec 10).
- Alta de sedes nuevas.
- Recargos / pagos parciales de colegiatura.

## Tareas

1. Crear `ViewModels/AddStudentViewModel.swift` con campos bindables, `var errorMatricula: String?`, `var errors: [Campo: String]`.
2. `func proposeProximoPago()` usando `CadenciaCobro.avanzar`.
3. `func proposeFechaCertificacion()` = inscripción + 12 semanas.
4. `func validate(existingMatriculas: Set<String>) -> Bool`:
   - Nombre trim no vacío.
   - Teléfono contacto: al menos 10 dígitos (filtrar no dígitos para contar).
   - Matrícula trim no vacía y no en `existingMatriculas`.
   - Sede en `SedeCatalog.all`.
   - Abono ≥ 0.
5. `func buildAlumno() -> Alumno` mapeando checkboxes a flags y abono clamp.
6. `Views/AddStudent/AddStudentView.swift`: `Form` agrupado:
   - Sección Datos
   - Sección Fechas y cadencia
   - Sección Pagos (checkboxes + abono + fecha certificación)
   - Botón Guardar
7. Mensajes de error bajo el campo: «Obligatorio», «Esta matrícula ya existe».
8. Cablear `navigationDestination` / `sheet` desde `DashboardView`. Tras save exitoso, `dismiss()`; el `@Query` debe mostrar el alumno (RF-006).
9. Tests `AddStudentViewModelTests`:
   - Guardar vacío → `validate` false, sin construir alumno válido.
   - Matrícula duplicada → error en matrícula.
   - Cadencia semanal, inscripción 01/03/2026, sin override → próximo pago 08/03/2026.
   - Inscripción 01/03/2026 → certificación 24/05/2026.
   - Checkboxes matrícula+guía → al construir, esos flags true.

## Criterios de aceptación

- Desde el dashboard, Nuevo alumno abre el formulario.
- Guardar sin obligatorios no inserta y marca campos.
- Matrícula repetida no inserta.
- Alta sin pagos: aparece en lista con pagado `$0.00` / total `$11,549.00` (o `$11,549`).
- Alta con Matrícula y Guía marcadas: pagado `$950.00`, saldo `$10,599.00`.
- Cambiar cadencia actualiza próximo pago propuesto.
- Fecha certificación default = inscripción + 12 semanas y es editable antes de guardar.
- Cancelar (back) no inserta.
- Tests de ViewModel en verde.

## Notas técnicas

- Teléfonos: persistir lo capturado (puede ser 10 dígitos locales). El prefijo `52` es spec 09.
- No usar `Double` para el TextField de abono: parsear `Decimal` / centavos.
- `fechaInscripcion` y date pickers en `.compact` / `.graphical` a criterio; mostrar solo fecha, no hora, excepto que `fechaProximoPago` pueda guardar noon local para evitar off-by-one.
- Unicidad: consultar `alumnos.map(\.matricula)` del Query padre o `FetchDescriptor` en el ViewModel.
- Si `#Unique` de SwiftData lanza al insertar duplicado, capturar y mostrar el mismo error de matrícula.
