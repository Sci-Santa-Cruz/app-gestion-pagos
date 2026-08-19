# Spec 04 — Student list and filters

## Objetivo

Debajo de los KPIs, mostrar la lista de alumnos con deuda, próximo pago y acciones, más buscador, filtro por sede, filtro por estado y ordenamiento, todos verificables.

## Contexto y dependencias

- **Fuente de verdad:** PRD RF-014, RF-015, RF-016, RF-017, RF-018.
- **Debe existir:** spec 02 (`Alumno`, `SedeCatalog`, `EstadoPago`, `Formatters`) y spec 03 (`DashboardView`, `StudentListSlot`, KPIs arriba).
- Las acciones Editar / Borrar / WhatsApp / Alarma deben **verse** en cada fila. Su comportamiento real es specs 07, 08 y 09. En esta spec: callbacks no-op o `assertionFailure` evitado; botones habilitados que no crashean.

Reglas de lista (completas):

- **Búsqueda** (case insensitive, substring): `nombreCompleto`, `matricula`, `telefonoContacto`, `telefonoAlterno`.
- **Filtro sede:** «Todas» o un valor de `SedeCatalog.all`.
- **Filtro estado:** Todos | Al día | Vencido | Con saldo | Liquidado.
  - Al día = `estadoPago == .alDia`
  - Vencido = `.vencido`
  - Con saldo = `saldo > 0`
  - Liquidado = `.liquidado`
- **Orden:**
  - Próximo pago (más cercano primero; nil al final)
  - Nombre A–Z (`localizedStandardCompare`)
  - Saldo mayor primero
  - Fecha inscripción más reciente primero
- Filtros se combinan con AND. Búsqueda se aplica sobre el conjunto ya filtrado (o viceversa; el resultado debe ser la intersección).

Fila (mínimo visible): nombre, matrícula, sede, pagado vs total (`Formatters.currency` ambos), saldo, fecha próximo pago (`dd/MM/yyyy` de `fechaProximoPagoEfectiva`), badge de estado, botones Editar, Borrar, WhatsApp, Alarma.

## Alcance

### In scope

- `StudentListView` + `StudentRowView` + `StudentListViewModel`.
- Reemplazar `StudentListSlot` del dashboard por esta lista.
- Search bar, pickers/menus de sede, estado y orden.
- Empty state de filtros: «Ningún alumno coincide».
- Botones de acción visibles con API de callbacks para specs posteriores.

### Out of scope

- Persistencia de filtros entre launches (no requerido).
- Implementar edición, borrado, notificaciones o WhatsApp.
- Cambiar fórmulas de KPI (siguen sobre **todos** los alumnos, no sobre la lista filtrada). **Importante:** los KPIs del dashboard ignoran búsqueda/filtros; solo la lista se filtra.

## Tareas

1. Crear `ViewModels/StudentListViewModel.swift`:
   - Props: `alumnos: [Alumno]`, `query: String`, `sede: String?` (nil = todas), `estadoFiltro: EstadoFiltro`, `orden: OrdenLista`.
   - `enum EstadoFiltro { case todos, alDia, vencido, conSaldo, liquidado }`
   - `enum OrdenLista { case proximoPago, nombre, saldo, fechaInscripcion }`
   - `var visibles: [Alumno]` aplica búsqueda + filtros + sort.
   - Extraer `normalized(_ s: String) -> String` folding diacríticos opcional; mínimo `lowercased()`.
2. Crear `Views/Dashboard/StudentRowView.swift`:
   - Layout compacto para iPhone (tarjeta o `VStack` en `List`).
   - Badge color: al día verde, vencido rojo, liquidado gris/azul.
   - Cuatro botones con SF Symbols sugeridos: `pencil`, `trash`, `message`, `bell`. Labels accesibles en español.
   - `struct StudentRowActions` con closures `onEdit`, `onDelete`, `onWhatsApp`, `onAlarm`.
3. Crear `Views/Dashboard/StudentListView.swift`: `.searchable`, toolbar o header con:
   - `Picker` sede: «Todas» + `SedeCatalog.all`
   - `Picker` estado con los 5 casos
   - `Picker` o menú orden
4. En `DashboardView`, sustituir `StudentListSlot` por `StudentListView(alumnos: alumnos)`. Pasar closures vacíos `{ _ in }` y comentarios `// spec 07/08/09`.
5. Tests `StudentListViewModelTests` (in-memory, sin UI):
   - Query `"555"` coincide teléfono o matrícula o nombre.
   - Filtro sede X + estado Vencido → solo esa intersección.
   - Orden nombre A–Z estable.
   - Orden saldo descendente.
   - Query que no pega → `visibles` vacío.
6. Preview: reutilizar los 3 alumnos de spec 03 y uno extra en otra sede.

## Criterios de aceptación

- Con alumnos en el store, la lista aparece **debajo** de los KPIs.
- Cada fila muestra nombre, matrícula, sede, pagado/total, saldo, próximo pago, estado y 4 acciones.
- Escribir en el buscador filtra en caliente (cada cambio de `query`).
- «Todas» muestra todas las sedes; elegir una sede oculta las demás.
- Filtro Vencido no muestra liquidados ni al día.
- KPIs **no** cambian al filtrar la lista.
- Tests de ViewModel en verde.
- Pulsar Editar/Borrar/WhatsApp/Alarma no crashea (no-op).

## Notas técnicas

- Usar `List` + `ScrollView` del dashboard: si el dashboard ya es `ScrollView`, no anidar `List` con scroll infinito raro; preferir `List` como contenedor principal del dashboard (KPIs en `Section` header / `safeAreaInset`) **o** `LazyVStack` dentro de un solo `ScrollView`. Elegir **un** scroll.
- No filtrar con SQL: el volumen PRD es cientos de alumnos; filtrar in-memory está bien.
- `fechaProximoPago` de la fila = `alumno.fechaProximoPagoEfectiva`.
- No implementar `.swipeActions` de borrar (RNF-007: el borrado real irá con confirmación en spec 07, no swipe único).
