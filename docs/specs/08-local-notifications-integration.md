# Spec 08 — Local notifications integration

## Objetivo

Programar, reemplazar y cancelar notificaciones locales de próximo cobro (la “alarma” del PRD), con permiso iOS y sin usar la app Reloj.

## Contexto y dependencias

- **Fuente de verdad:** PRD sección alarmas, RF-022, RF-023, RF-024, RF-025, RNF-006, RNF-008.
- **Debe existir:** spec 05/07 (guardar alumno, botón Alarma en la fila — hoy no-op), spec 02 (`fechaProximoPago`, `alarmaHour`, `alarmaMinute`, saldo, nombre, matrícula).
- iOS **no** permite crear alarmas del Reloj. API: `UserNotifications` / `UNUserNotificationCenter`.
- Identifier estable: `cobro.\(alumno.id.uuidString)` — una notificación por alumno.

Comportamiento:

1. Primer uso: pedir permiso. Si deniega, guardar igual y mostrar aviso no bloqueante («No se programó el recordatorio. Actívalo en Ajustes.»).
2. Al **guardar** alta o edición con `fechaProximoPago` y saldo > 0: schedule para esa fecha a `alarmaHour:alarmaMinute` (default 09:00). Cancelar la previa del mismo id.
3. Si el alumno queda **liquidado** al guardar: cancelar, no schedule.
4. Si se **borra**: cancelar.
5. Botón **Alarma** en la fila: sheet/datepicker fecha **y** hora → actualiza `fechaProximoPago` + hora → reprograma. Texto de ayuda: «Recordatorio de la app (no es la alarma del iPhone)».
6. Cuerpo de la notificación: nombre, matrícula, saldo formateado. Título: «Cobro pendiente». Ejemplo cuerpo: `{nombre} · {matricula} · saldo {currency}`.

Trigger: `UNCalendarNotificationTrigger` (date components locales, `repeats: false`).

## Alcance

### In scope

- `Services/NotificationService.swift` (protocol + implementación real + mock para tests).
- Pedido de permiso.
- Integración en save add/edit, delete, y botón Alarma.
- UI de reprogramar (fecha + hora).
- Info.plist usage string (ya en spec 01; verificar que exista).

### Out of scope

- App Reloj, EventKit, Reminders.
- Notificaciones push (APNs).
- Recurring every week (una fecha a la vez).
- Acciones en la notificación (botones de la notificación no requeridos).
- Banner in-app si la app está foreground (opcional; no bloquea).

## Tareas

1. Protocolo:

```swift
protocol NotificationScheduling {
    func requestAuthorization() async -> Bool
    func scheduleCobro(alumnoID: UUID, nombre: String, matricula: String, saldoText: String, date: Date) async
    func cancelCobro(alumnoID: UUID) async
}
```

2. `NotificationService`: `UNUserNotificationCenter.current()`. Content `sound: .default`. Si `date < now`, no programar (o programar +1 min solo si el usuario acaba de elegir hora pasada hoy: **no programar en el pasado**; mostrar «Elige una fecha futura»).
3. `NotificationService` no-op mock para tests de ViewModel.
4. Inyectar en `StudentFormViewModel.save` y en delete del dashboard.
5. `Views/Dashboard/AlarmPickerSheet.swift`: `DatePicker` displayedComponents `[.date, .hourAndMinute]`. Confirmar → write alumno + schedule.
6. Cablear `onAlarm` de spec 04.
7. Al iniciar app (opcional ligero): no reschedule masivo obligatorio en esta spec; save/alarm/delete bastan. Si se desea robustez, `App` al `task` puede pedir permiso sin spamear: solo si aún `.notDetermined` y hay alumnos con saldo.
8. Tests con mock:
   - Save alumno con próximo pago futuro → `schedule` llamado 1 vez con id correcto.
   - Save de nuevo → `cancel` + `schedule` o `schedule` que reemplaza (mismo identifier; UNUserNotification reemplaza). Afirmar una sola pending en la implementación real si se usa centro in-test (puede ser frágil); con mock, `scheduleCallCount` y mismo id.
   - Liquidar (todos pagados) → `cancel` y no `schedule`.
   - Delete → `cancel`.
   - Permiso false → save no lanza; flag `didWarnPermissionDenied` true.

## Criterios de aceptación

- Con permiso OK, alta con próximo pago mañana 09:00 deja una `UNNotificationRequest` pendiente con ese calendario (verificar en debug imprimiendo `getPendingNotificationRequests` o test de integración en simulador).
- Reprogramar con Alarma deja **una** request de ese alumno y la fila muestra la nueva fecha.
- Borrar o liquidar: `getPending` ya no contiene `cobro.<id>`.
- Permiso denegado: el alumno se guarda; aparece aviso; la app no se bloquea (RNF-008).
- Ningún código usa `ClockKit` ni URL `clock-alarm://`.
- Copy visible de que es recordatorio de Nova Impulsa, no alarma del Reloj.

## Notas técnicas

- Entitlement no se necesita para locales.
- `UNCalendarNotificationTrigger(dateMatching: DateComponents(year,month,day,hour,minute), repeats: false)`.
- TimeZone: `Calendar.current.timeZone`.
- Main actor: el centro es thread-safe; ViewModels `@MainActor`.
- Si el usuario abre Ajustes y concede permiso después, no hay reschedule automático en esta spec (aceptable). El botón Alarma puede reintentar permiso.
- Foreground: implementar `UNUserNotificationCenterDelegate.willPresent` con `[.banner, .sound]` para poder demo en simulador con app abierta.
