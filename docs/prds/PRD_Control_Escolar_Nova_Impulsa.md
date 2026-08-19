# PRD — Control Escolar y Cobranza
## Nova Impulsa

**Producto:** app iOS interna (iPhone)
**Versión:** primer entregable (alcance completo)
**Fecha:** 18 de agosto de 2026

---

## Contexto y problema

Nova Impulsa opera un programa escolar de aproximadamente 12 semanas. Hoy el control de alumnos y la cobranza se hacen sin una herramienta dedicada: no hay un vistazo único de quién está al día, quién debe, cuánto falta por cobrar ni cuándo hay que dar seguimiento.

El costo total por alumno es de **$11,549**, compuesto por cargos fijos y tres colegiaturas. La certificación/gestoría debe liquidarse en la semana 12. Sin un registro único, es fácil perder fechas de cobro, duplicar seguimientos o no detectar cartera vencida.

Se requiere una app interna, exclusiva para iPhone, que concentre el padrón, el estado de pagos y las acciones de cobranza (editar, borrar, WhatsApp y recordatorio).

---

## Objetivo

Entregar en un solo release una app iPhone de uso interno que permita dar de alta, editar y borrar alumnos; registrar su plan de pagos (fijos, colegiaturas y abonos a certificación); consultar en un dashboard la cartera (pendientes, al día, vencidos, cobrado, por cobrar y certificación pendiente); buscar y filtrar; contactar por WhatsApp; programar recordatorios locales de próximo cobro; e importar/exportar la información en CSV compatible con Excel.

---

## Público objetivo y usuarios

- **Usuario único:** el operador del negocio (dueño o persona que administra inscripción y cobranza).
- **No hay roles, permisos ni login.** Sede y Asesor son datos del alumno, no cuentas de la app.
- **Dispositivo:** iPhone. Un dispositivo, una base local. No hay sync entre iPhones.
- **Alumno:** no es usuario de la app; es el registro sobre el que se cobra y se da seguimiento.

---

## Alcance

### In scope

- Alta, edición y borrado permanente de alumnos.
- Captura de: nombre completo, teléfono de contacto, teléfono alterno, matrícula, sede, asesor, fecha de inscripción, cadencia de cobro y fecha de próximo pago.
- Plan de pagos del alumno: Matrícula, Guía, Colegiaturas 1, 2 y 3 (marcado pagado / no pagado) y Certificación/Gestoría con abonos.
- Cálculo automático de total esperado, pagado, saldo, próximo pago y estado (al día / vencido / liquidado).
- Dashboard con KPIs, buscador, filtros (sede, estado), ordenamiento y lista de alumnos con deuda, próximo pago y acciones.
- Acciones por alumno: Editar, Borrar, WhatsApp, Alarma (recordatorio local).
- Importar y exportar alumnos y pagos en CSV compatible con Excel.
- Persistencia local en el iPhone.
- Permiso y programación de notificaciones locales.

### Out of scope

- Android, iPad, web o backend.
- Multiusuario, autenticación, roles o varios operadores.
- Sincronización iCloud/CloudKit o entre dispositivos.
- App Reloj (alarmas del sistema), Calendar o Recordatorios de Apple.
- WhatsApp Business API; solo apertura del chat nativo.
- Archivo `.xlsx` nativo (Excel se cubre con CSV).
- Overwrite o merge en importación; no se actualizan alumnos existentes.
- Baja reversible / desactivación (solo borrado permanente).
- Recargos, descuentos, prorrateo o intereses.
- Pagos parciales de Matrícula, Guía o Colegiaturas (solo certificación admite abonos).
- Dividir las colegiaturas en 12 cargos semanales o 6 quincenales.
- Múltiples programas/certificaciones por alumno.
- Control de asistencia, documentos, fotos o catálogos administrables (pantallas para dar de alta sedes/asesores).
- Reportes avanzados más allá del dashboard y del CSV.

---

## Conceptos de dominio

### Sede

Valor descriptivo del lugar donde se atiende al alumno. Se elige de una **lista fija en la app** (no texto libre, no entidad administrable). Sirve para filtrar el dashboard. Un alumno tiene una sola sede.

### Asesor

Nombre de la persona interna que da seguimiento al alumno. Es un **dato de texto** asociado al alumno, no un usuario del sistema. No hay permisos por asesor.

### Alumno

Persona inscrita al programa. Identificador de negocio: **matrícula**, única en el dispositivo. Estado operativo: activo hasta que se borra (no hay baja lógica).

### Cadencia de cobro

Frecuencia de seguimiento: **semanal**, **quincenal** o **mensual**. No cambia el monto ni el número de cargos. Solo propone la **fecha de próximo pago** (intervalo de 7 días, 15 días o 1 mes calendario a partir de la inscripción). El operador puede sobrescribir esa fecha.

### Estructura de costos: fijos vs. colegiaturas vs. certificación

Costo total esperado por alumno (si no se editan montos): **$11,549**.

| Tipo | Concepto | Importe | Comportamiento |
|------|----------|---------|----------------|
| Fijo | Matrícula | $700 | Binario: pagado o no. Vence en la fecha de inscripción. |
| Fijo | Guía | $250 | Binario: pagado o no. Vence en la fecha de inscripción. |
| Variable en fecha, fijo en monto | Colegiatura 1, 2 y 3 | $1,200 c/u ($3,600) | Tres cargos binarios. Vencen en inscripción + 1, 2 y 3 meses. La cadencia no los parte. |
| Fijo en tope, variable en abonos | Certificación / Gestoría | $6,999 | Único concepto con **abonos**. Fecha programada = inscripción + 12 semanas, editable. Liquidado cuando lo abonado ≥ $6,999. |

Montos por defecto editables al alta o en edición (por si un caso puntual difiere). El total esperado de ese alumno es la suma de sus importes vigentes, no necesariamente $11,549.

### Cálculo de deuda

Por alumno:

- **Total esperado** = suma de importes de Matrícula + Guía + Colegiaturas 1–3 + $6,999 de certificación (o el tope editado).
- **Total pagado** = suma de conceptos binarios marcados como pagados + suma de abonos a certificación (tope: no cuenta por encima del importe de certificación).
- **Saldo / deuda** = total esperado − total pagado.
- **Liquidado** = saldo = 0.

Agregados del dashboard (solo alumnos existentes en el dispositivo):

- **Alumnos con pagos pendientes:** cantidad con saldo > 0.
- **Alumnos al día:** saldo > 0 y ningún cargo vencido.
- **Alumnos vencidos:** al menos un cargo no cubierto con fecha programada < hoy.
- **Ingresos cobrados:** suma de totales pagados.
- **Por cobrar total:** suma de saldos.
- **Certificación pendiente:** suma de (importe certificación − abonado) de todos los alumnos.

**Cargo vencido:**

- Matrícula o Guía no pagadas y fecha de inscripción < hoy.
- Colegiatura N no pagada y (inscripción + N meses) < hoy.
- Certificación con restante > 0 y su fecha programada < hoy.

**Próximo pago (lista):** la fecha operativa de seguimiento del alumno (`fecha_proximo_pago`). Si está vacía, se usa la fecha programada más próxima entre cargos aún no cubiertos.

### Alarmas nativas en iOS (recordatorio local)

iOS **no permite** crear alarmas en la app Reloj. En este producto, “alarma nativa” significa **notificación local** (`UNUserNotificationCenter`):

1. Al guardar un alumno con `fecha_proximo_pago`, se programa una notificación para esa fecha (hora por defecto 09:00, editable al agendar/reprogramar).
2. El identificador de la notificación es estable por alumno; un alta o edición **reemplaza** la anterior.
3. Si el alumno queda liquidado o se borra, la notificación se **cancela**.
4. El botón **Alarma** en la lista abre un selector de fecha/hora y reprograma esa notificación.
5. Si el usuario niega el permiso de notificaciones, el resto de la app funciona; se informa que el recordatorio no se programará.
6. El cuerpo de la notificación incluye nombre, matrícula y monto de saldo.

---

## Requerimientos funcionales

**Alumnos**

- **RF-001.** El sistema debe permitir dar de alta un alumno con: nombre completo (obligatorio), teléfono de contacto (obligatorio), teléfono alterno (opcional), matrícula (obligatorio, único), sede (obligatorio, catálogo fijo), asesor (opcional), fecha de inscripción (obligatorio, default hoy), cadencia (obligatorio: semanal / quincenal / mensual) y fecha de próximo pago (obligatorio).
- **RF-002.** Si falta un campo obligatorio o la matrícula ya existe, el sistema debe impedir guardar y mostrar el error en el campo correspondiente.
- **RF-003.** Al cambiar cadencia o fecha de inscripción, el sistema debe proponer `fecha_proximo_pago` = inscripción + 7 días, + 15 días o + 1 mes, sin impedir que el usuario la edite.
- **RF-004.** El sistema debe permitir editar todos los datos de un alumno existente, incluyendo marcados de pago y abonos, y persistir los cambios al guardar.
- **RF-005.** El sistema debe permitir borrar un alumno de forma permanente, junto con todos sus cargos y abonos, solo después de una confirmación que muestre el nombre y la matrícula.
- **RF-006.** Tras el alta, la edición o el borrado, el alumno debe aparecer, actualizarse o desaparecer de la lista y de los KPIs sin reiniciar la app.

**Pagos**

- **RF-007.** Al alta, el sistema debe crear exactamente estos cargos por defecto: Matrícula $700, Guía $250, Colegiatura 1 $1,200, Colegiatura 2 $1,200, Colegiatura 3 $1,200, Certificación/Gestoría $6,999.
- **RF-008.** En el formulario de alta y de edición, el sistema debe mostrar checkboxes para Matrícula, Guía y Colegiaturas 1, 2 y 3; marcar el checkbox registra el concepto como pagado al 100% de su importe.
- **RF-009.** El sistema debe permitir capturar abonos a Certificación (importe > 0). El acumulado no debe superar el importe del concepto; al alcanzar el tope, el concepto queda liquidado.
- **RF-010.** El sistema debe asignar fecha programada de certificación = fecha de inscripción + 12 semanas, y permitir editarla.
- **RF-011.** El sistema debe calcular y mostrar por alumno: total esperado, total pagado, saldo y estado (Al día, Vencido o Liquidado), según las reglas de dominio de este PRD.
- **RF-012.** Desmarcar un checkbox de concepto binario debe revertirlo a no pagado y recalcular saldo, estado, KPIs y próximo pago.

**Dashboard**

- **RF-013.** La pantalla principal debe mostrar los KPIs: alumnos con pagos pendientes, alumnos al día, alumnos vencidos, ingresos cobrados, por cobrar total y certificación pendiente, calculados con las definiciones de dominio.
- **RF-014.** La lista debe mostrar por alumno al menos: nombre, matrícula, sede, pagado vs. total, saldo, fecha de próximo pago, estado y acciones Editar, Borrar, WhatsApp y Alarma.
- **RF-015.** El buscador debe filtrar en tiempo de escritura por coincidencia parcial (sin distinguir mayúsculas) en nombre, matrícula o cualquiera de los dos teléfonos.
- **RF-016.** El sistema debe filtrar por sede (incluida la opción “Todas”).
- **RF-017.** El sistema debe filtrar por estado: Todos, Al día, Vencido, Con saldo, Liquidado.
- **RF-018.** El sistema debe ordenar la lista por: próximo pago (más cercano primero), nombre (A–Z), saldo (mayor primero) o fecha de inscripción (más reciente primero).
- **RF-019.** La pantalla principal debe ofrecer un acceso directo para abrir el formulario de nuevo alumno.

**WhatsApp y alarma**

- **RF-020.** Al pulsar WhatsApp, el sistema debe abrir `https://wa.me/<telefono_contacto>` con el teléfono de contacto en formato internacional (si el número tiene 10 dígitos, prefijo `52`). Si WhatsApp no está instalado, debe mostrar un mensaje de error y no fallar en silencio.
- **RF-021.** El chat debe prellenar el mensaje: `Hola {nombre}, te contactamos de Nova Impulsa por tu próximo pago del {fecha_proximo_pago}. Saldo: ${saldo}.`
- **RF-022.** Al guardar un alumno con fecha de próximo pago, el sistema debe programar una notificación local para esa fecha a las 09:00 (o la hora elegida en RF-023), cancelando cualquier notificación previa de ese alumno.
- **RF-023.** El botón Alarma debe permitir elegir fecha y hora y reprogramar la notificación local de ese alumno, actualizando `fecha_proximo_pago`.
- **RF-024.** Si el alumno se liquida o se borra, el sistema debe cancelar su notificación local.
- **RF-025.** En el primer uso de alarmas, el sistema debe pedir permiso de notificaciones. Si se deniega, debe avisarlo y continuar sin bloquear alta, edición ni cobranza.

**Importar / Exportar / Excel**

- **RF-026.** Exportar y Excel deben generar el mismo archivo CSV (UTF-8, coma) y abrirlo en la hoja de compartir de iOS. Columnas mínimas: matrícula, nombre, teléfono, teléfono_alterno, sede, asesor, fecha_inscripcion, cadencia, fecha_proximo_pago, matricula_pagada (si/no), guia_pagada, col1_pagada, col2_pagada, col3_pagada, certificacion_abonado, certificacion_importe, certificacion_fecha, total_esperado, total_pagado, saldo, estado.
- **RF-027.** Importar debe leer un CSV con esas columnas (las de totales/estado pueden omitirse; se recalculan). Solo debe crear alumnos **nuevos**.
- **RF-028.** Si una fila tiene matrícula ya existente o datos obligatorios inválidos, el sistema no debe crear ese alumno; debe continuar con el resto y mostrar un resumen: filas OK, filas con error y motivo.
- **RF-029.** La importación no debe modificar ni borrar alumnos ya guardados.

---

## Requerimientos no funcionales

- **RNF-001.** Solo iPhone. Sin soporte oficial para iPad.
- **RNF-002.** Swift + SwiftUI. Persistencia local con SwiftData. iOS 17 o superior.
- **RNF-003.** Sin backend, sin cuenta de usuario y sin sync en la nube de la app. El respaldo de negocio es el CSV exportado.
- **RNF-004.** Debe funcionar offline, excepto WhatsApp.
- **RNF-005.** UI en español de México. Moneda `MXN` con `$` y dos decimales. Fechas `dd/MM/yyyy`.
- **RNF-006.** Desempeño fluido con al menos 500 alumnos y 3,000 cargos en un iPhone reciente.
- **RNF-007.** El borrado permanente no puede ser un swipe único sin confirmación.
- **RNF-008.** Si el permiso de notificaciones está denegado, ninguna otra función debe interrumpirse.
- **RNF-009.** Los importes y fechas no deben perder precisión al persistir (decimales de dinero a 2 dígitos).

---

## Criterios de aceptación del MVP

**Alta**

- Dado el formulario vacío, cuando el usuario guarda sin nombre, teléfono de contacto, matrícula, sede, cadencia o fecha de próximo pago, entonces no se crea el alumno y se marca el campo faltante.
- Dado una matrícula ya usada, cuando se intenta un alta nueva con esa matrícula, entonces se rechaza el guardado.
- Dado cadencia semanal e inscripción 01/03/2026, cuando el usuario no toca la fecha de próximo pago, entonces esa fecha queda en 08/03/2026 y es editable.
- Dado un alta con checkboxes vacíos y abono $0, cuando se guarda, entonces el saldo es $11,549 y el estado no es Liquidado.
- Dado Matrícula y Guía marcadas al alta, cuando se guarda, entonces total pagado = $950 y saldo = $10,599.

**Certificación y vencimiento**

- Dado inscripción 01/03/2026, cuando se guarda, entonces la fecha de certificación es 24/05/2026 (12 semanas) y es editable.
- Dado un abono de $2,000 a certificación, cuando se consulta el alumno, entonces certificación pendiente de ese alumno es $4,999 y el saldo baja $2,000.
- Dado un abono que excedería $6,999, cuando se captura, entonces se rechaza o se recorta al restante; el acumulado nunca supera el tope.
- Dado Colegiatura 1 no pagada e inscripción hace más de un mes, cuando se abre el dashboard, entonces ese alumno cuenta en **Vencidos** y no en **Al día**.

**Dashboard**

- Dados 2 alumnos con saldo, 1 de ellos vencido y 1 liquidado, cuando se abre el dashboard, entonces pendientes = 2, vencidos = 1, al día = 1 (el no vencido con saldo), ingresos cobrados = suma de pagado, por cobrar = suma de saldos.
- Dado texto “555”, cuando se busca, entonces se listan alumnos cuyo nombre, matrícula o teléfono contiene 555.
- Dado filtro sede y estado Vencido, cuando se aplican, entonces solo aparecen alumnos de esa sede en estado vencido.

**Editar / borrar**

- Dado un alumno, cuando se edita el nombre y se marca Colegiatura 1, entonces la lista muestra el nuevo nombre y el saldo disminuye $1,200.
- Dado confirmar borrar, cuando se acepta, entonces el alumno desaparece de lista y KPIs y no puede recuperarse en la app.
- Dado el diálogo de borrar, cuando se cancela, entonces no cambia nada.

**WhatsApp y alarma**

- Dado teléfono 5512345678, cuando se pulsa WhatsApp, entonces se abre wa.me con `525512345678` y el mensaje prellenado con nombre, fecha y saldo.
- Dado guardar un alumno con próximo pago mañana a las 09:00 y permiso concedido, entonces existe una notificación local programada para ese instante.
- Dado el botón Alarma, cuando se elige otra fecha/hora, entonces queda una sola notificación (la nueva) y la lista muestra esa fecha.
- Dado borrar o liquidar al alumno, entonces ya no hay notificación pendiente de ese alumno.
- Dado permiso denegado, cuando se guarda un alumno, entonces la ficha se guarda y se informa que no habrá recordatorio.

**CSV**

- Dado al menos un alumno, cuando se pulsa Exportar o Excel, entonces se obtiene un CSV con las columnas de RF-026 y los importes coinciden con la pantalla.
- Dado un CSV válido de 3 alumnos nuevos, cuando se importa, entonces se crean 3 registros y los KPIs aumentan en consecuencia.
- Dado un CSV con 1 matrícula duplicada y 2 nuevas, cuando se importa, entonces se crean 2, se omite 1 y el resumen indica la matrícula rechazada.
- Dado un alumno ya existente, cuando se reimporta su fila, entonces no se altera su saldo ni sus pagos.

El primer entregable se acepta solo si **todos** los RF anteriores son demostrables en un iPhone (simulador o dispositivo).

---

## Riesgos y supuestos

### Riesgos

- Sin nube, perder o cambiar de iPhone implica perder datos si no se exportó CSV.
- Las notificaciones locales pueden no verse (permiso denegado, No Molestar, modo concentración). El estado de cobranza en pantalla sigue siendo la fuente de verdad.
- “Alarma” puede esperarse como app Reloj; hay que dejar claro en UI que es un recordatorio de la app.
- Checkboxes binarios en colegiaturas no modelan pagos semanales de $300; si operación cobra fracciones de $1,200, el saldo quedará mal hasta un cambio de producto.
- Importar CSV mal formado puede generar muchos errores de fila; el resumen debe ser legible.
- WhatsApp ausente o número mal capturado impide el contacto rápido.
- Borrado permanente no tiene papelera.

### Supuestos

- Un alumno cursa un solo programa a la vez.
- La lista de sedes es corta y estable; se mantiene en código. Los valores concretos los define negocio antes de release (no bloquean el modelo).
- El operador usa un solo iPhone como sistema de registro.
- WhatsApp está instalado en el dispositivo de uso diario.
- $11,549 y los montos de la tabla son los default; excepciones se resuelven editando importes en el alumno.
- “Excel” significa CSV que Excel abre, no generación de `.xlsx`.
- La hora por defecto del recordatorio (09:00) es aceptable para cobranza matutina.
- Volumen esperado: cientos de alumnos, no miles.
