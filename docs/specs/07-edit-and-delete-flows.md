# Spec 07 — Edit and delete flows

## Objetivo

Permitir editar todos los datos y pagos de un alumno existente y borrarlo de forma permanente con confirmación explícita, actualizando lista y KPIs sin reiniciar la app.

## Contexto y dependencias

- **Fuente de verdad:** PRD RF-004, RF-005, RF-006, RF-012, RNF-007.
- **Debe existir:** spec 04 (callbacks `onEdit` / `onDelete` en la fila), spec 05 (formulario de alta), spec 06 (`PaymentCalculator`).
- **Aún no:** cancelar notificación al borrar/liquidar (spec 08 lo enganchará). En esta spec, borrar solo elimina SwiftData. Dejar un hook `StudentLifecycle` protocol o comentario `// spec 08: cancel notification` en el delete para no pelear el merge.
- No hay baja reversible. Borrar = desaparece el registro y no hay papelera.

Confirmación de borrado (RNF-007):

- No swipe-to-delete como única vía.
- Alert: título «¿Borrar alumno?», mensaje con **nombre** y **matrícula**, botones «Cancelar» y «Borrar» (rol destructivo).
- Cancelar: no muta el store.

Edición:

- Reutilizar el mismo formulario visual que el alta (`StudentFormView` extraído si spec 05 dejó todo dentro de `AddStudentView`).
- Matrícula editable pero sigue única (conflicto con **otro** alumno).
- Checkboxes y abono editables; desmarcar recalcula (RF-012).
- Guardar persisté y `dismiss`; lista y KPIs reflejan cambios (RF-006) vía `@Query`.

## Alcance

### In scope

- Extraer formulario compartido si no existe.
- `EditStudentView` / modo `FormMode.add | .edit(Alumno)`.
- Cablear `onEdit` de la fila → push/sheet de edición.
- Cablear `onDelete` → alert → `modelContext.delete` + `save`.
- Validaciones iguales al alta.
- Tests de ViewModel: duplicado de matrícula al editar, desmarcar col1 reduce saldo 1200, delete no se ejecuta si cancel.

### Out of scope

- Undo / papelera / inactivo.
- Notificaciones (spec 08).
- CSV (spec 10).
- Borrado masivo.

## Tareas

1. Extraer `Views/StudentForm/StudentFormView.swift` + `ViewModels/StudentFormViewModel.swift` (renombrar el de alta). Init `mode: add` o `edit(Alumno)` precarga campos.
2. Al guardar en edit: escribir sobre el mismo `Alumno` (no insertar uno nuevo). Clamp abono. Recalcular `fechaCertificacion` **solo si** el usuario cambió inscripción y no había override de fecha cert; si es más simple, no auto-cambiar certificación en edit salvo que el DatePicker la cambie (documentar: en edit, fechas no se pisan solas).
3. En add, mantener el comportamiento de spec 05 (propuesta automática).
4. `DashboardView` / lista:
   - `onEdit`: `navigationPath.append(.edit(alumno.id))` o sheet.
   - `onDelete`: set `alumnoPendienteBorrado` → `.alert`.
5. Tras delete, el alumno no debe quedar en memoria de filtros.
6. Tests:
   - Edit nombre + `col1Pagada true` → `nombre` nuevo y saldo original − 1200.
   - Edit matrícula a una ya usada por otro → error, store intacto.
   - Simular confirmación delete → fetch count − 1.
7. Preview de edición con alumno de muestra.

## Criterios de aceptación

- Editar en la fila abre el formulario con datos precargados (nombre, teléfonos, pagos).
- Guardar cambio de nombre se ve en la lista al instante.
- Marcar Colegiatura 1 en edición baja el saldo $1,200 y sube ingresos cobrados en el KPI.
- Desmarcar un concepto pagado sube el saldo otra vez.
- Alert de borrar muestra nombre y matrícula.
- Cancelar en el alert deja al alumno y los KPIs iguales.
- Confirmar borrar lo quita de lista y de KPIs; no se puede reabrir.
- No hay swipe-delete sin alert (si hay swipe, debe abrir el **mismo** alert, no borrar directo).
- Tests en verde.

## Notas técnicas

- Identificar alumnos por `id: UUID` en la navegación, no por índice de lista filtrada.
- `modelContext.delete(alumno)` borra el único modelo; no hay cascada porque no hay `Pago` hijo.
- Evitar copiar el `Alumno` a un struct, editar y perder la identity de SwiftData.
- Unicidad en edit: `existing.filter { $0.id != current.id }.map(\.matricula)`.
- Spec 08 buscará `StudentFormViewModel.save` y el método delete: exportar un `protocol NotificationScheduling` vacío o no-op default para que 08 solo inyecte el servicio.
