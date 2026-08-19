# Spec 03 — Dashboard UI and KPIs

## Objetivo

Construir la pantalla principal con los 6 KPIs de cartera, la barra de acciones Importar/Exportar/Excel (aún no funcionales) y el cascarón de navegación, usando los cálculos de `Alumno` ya persistidos.

## Contexto y dependencias

- **Fuente de verdad:** PRD RF-013, RF-019 (acceso a alta se deja listo; el formulario es spec 05), RNF-005.
- **Debe existir:** spec 01 (app SwiftUI) y spec 02 (`Alumno`, `EstadoPago`, `Formatters`, `SedeCatalog`).
- **Aún no existen:** lista filtrable (spec 04), formulario real (spec 05), CSV (spec 10). Esta spec deja placeholders para ellos.

Definiciones de KPI (repetidas para que esta spec sea suficiente):

| KPI | Cálculo sobre todos los `Alumno` del store |
|-----|--------------------------------------------|
| Alumnos con pagos pendientes | count `saldo > 0` |
| Alumnos al día | count `saldo > 0 && !estaVencido` |
| Alumnos vencidos | count `estaVencido` |
| Ingresos cobrados | sum `totalPagado` |
| Por cobrar total | sum `saldo` |
| Certificación pendiente | sum `certificacionPendiente` |

Un alumno liquidado (`saldo == 0`) no cuenta en pendientes, al día ni vencidos.

## Alcance

### In scope

- `DashboardView` como raíz de `WindowGroup` (reemplaza el placeholder de spec 01).
- `DashboardViewModel` (@Observable) que lee `[Alumno]` vía `@Query` o que recibe alumnos y expone los 6 KPIs.
- UI de 6 tarjetas de métrica, título, botones Importar / Exportar / Excel visibles.
- Estado vacío (0 alumnos): KPIs en 0 y mensaje «No hay alumnos».
- Botón flotante o equivalente «Nuevo alumno» visible; destination placeholder.
- `NavigationStack`.
- Preview con store in-memory y 3 alumnos de muestra para verificar números.

### Out of scope

- Buscador, filtros, sort y filas de alumno (spec 04). Dejar un `StudentListPlaceholder` vacío o `Color.clear` con frame mínimo para que 04 inserte la lista **debajo** de los KPIs.
- Lógica de import/export (botones no-op o `print`; no crashear).
- Editar, borrar, WhatsApp, alarma.
- Recalcular fórmulas distintas a spec 02.

## Tareas

1. Reemplazar `RootPlaceholderView` por `Views/Dashboard/DashboardView.swift` en `NovaImpulsaApp`.
2. Crear `ViewModels/DashboardViewModel.swift`:
   - Input: `[Alumno]`.
   - Output: `pendientesCount`, `alDiaCount`, `vencidosCount`, `ingresosCobrados: Decimal`, `porCobrar: Decimal`, `certificacionPendiente: Decimal`.
   - Usar `Date.now` (inyectable `now: Date` para tests).
3. Crear `Views/Dashboard/KPICard.swift`: título corto + valor. Montos con `Formatters.currency`. Conteos como entero.
   - Títulos exactos (español): «Pagos pendientes», «Al día», «Vencidos», «Ingresos cobrados», «Por cobrar», «Certificación pendiente».
4. Layout:
   - Header «Nova Impulsa» / subtítulo «Control escolar».
   - HStack/Wrap de botones: `Importar`, `Exportar`, `Excel` (style bordered). Acción vacía `{ }` y comentario `// spec 10`.
   - Grid 2×3 o scroll horizontal de KPIs; debe verse en iPhone SE y Pro sin recortar títulos.
   - Zona inferior reservada: `StudentListSlot()` — archivo `Views/Dashboard/StudentListSlot.swift` que por ahora muestra el empty state si no hay alumnos.
   - `safeAreaInset` o overlay: botón circular `+` / «Nuevo alumno». Navegar a `Text("Alta — spec 05")` con `.navigationDestination`.
5. `DashboardView` usa `@Query(sort: \Alumno.nombreCompleto) private var alumnos: [Alumno]` y pasa el array al ViewModel. Los KPIs deben actualizarse al cambiar el store (spec 05/07 lo verificarán; aquí basta Query).
6. Preview `#Preview` con `ModelContainer` in-memory:
   - Alumno A: defaults, nada pagado, inscripción hace 2 meses → vencido (col1), saldo 11549.
   - Alumno B: nada vencido, matrícula pagada, saldo > 0, `fechaInscripcion = today` → al día.
   - Alumno C: todo pagado (flags true y abono = 6999) → liquidado, saldo 0.
   - KPIs esperados en preview: pendientes = 2, vencidos = 1, al día = 1, ingresos = pagado(A)+pagado(B)+pagado(C), por cobrar = saldo(A)+saldo(B), cert pendiente = suma restantes.
7. Test `DashboardViewModelTests`: construir 2 con saldo (1 vencido) + 1 liquidado y afirmar pendientes=2, vencidos=1, alDia=1, porCobrar = suma saldos, ingresos = suma pagados.

## Criterios de aceptación

- Al abrir la app se ve el dashboard, no el placeholder «Nova Impulsa» suelto de spec 01 (el nombre puede seguir en el header).
- Con 0 alumnos, los 6 KPIs muestran 0 / $0.00 y el empty state.
- El test de ViewModel pasa con el escenario de 3 alumnos del PRD (2 con saldo, 1 vencido, 1 liquidado).
- Botones Importar, Exportar, Excel son visibles y pulsables sin crash.
- El control de nuevo alumno es visible.
- Montos con formato mexicano (`$11,549.00` o equivalente `es_MX`).
- No se implementa CSV ni lista filtrable.

## Notas técnicas

- No duplicar fórmulas de deuda: el ViewModel solo **agrega** propiedades de `Alumno`.
- iPhone only; usar `NavigationStack`, no `NavigationSplitView` de iPad.
- Accesibilidad: cada KPI con `accessibilityLabel` «\(título): \(valor)».
- Los botones CSV deben permanecer en esta barra para que spec 10 solo les asigne acción.
- Si `StudentListSlot` se reemplaza en spec 04, mantener KPIs arriba y botones CSV arriba de la lista.
