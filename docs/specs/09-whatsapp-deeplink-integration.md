# Spec 09 — WhatsApp deeplink integration

## Objetivo

Desde la fila del alumno, abrir el chat de WhatsApp con el teléfono de contacto en formato internacional y un mensaje prellenado de cobranza; si WhatsApp no está, mostrar error explícito.

## Contexto y dependencias

- **Fuente de verdad:** PRD RF-020, RF-021.
- **Debe existir:** spec 04 (`onWhatsApp` no-op) y spec 02 (teléfonos, nombre, `fechaProximoPago`, `saldo`).
- No hay WhatsApp Business API. Solo URL scheme / `https://wa.me/`.
- Spec 01 ya debió poner `LSApplicationQueriesSchemes` = `whatsapp`. Verificarlo; si falta, añadirlo **en esta spec**.

Reglas de número:

- Tomar `telefonoContacto` (no el alterno).
- Dejar solo dígitos.
- Si queda **10** dígitos → prefijo `52` (México).
- Si ya empieza por `52` y tiene 12 dígitos, no duplicar.
- Si tiene 11–15 dígitos y no es el caso 10, usar tal cual (operador pudo guardar LADA internacional).
- Si hay < 10 dígitos → no abrir; alert «Teléfono de contacto inválido».

Mensaje exacto (RF-021):

```
Hola {nombre}, te contactamos de Nova Impulsa por tu próximo pago del {fecha_proximo_pago}. Saldo: ${saldo}.
```

- `{nombre}` = `nombreCompleto`
- `{fecha_proximo_pago}` = `dd/MM/yyyy` de `fechaProximoPagoEfectiva`
- `{saldo}` = monto con formato `es_MX` (p. ej. `$10,599.00` o el mismo formatter de la app). El PRD muestra `Saldo: ${saldo}.` — usar el formatter de moneda completo, no inventar otro.

URL: `https://wa.me/<digits>?text=<percentEncoded>`

Detección instalado: `UIApplication.shared.canOpenURL(URL(string: "whatsapp://")!)` (requiere el scheme en Info.plist). Si false → alert «Instala WhatsApp para contactar al alumno.» No fallar en silencio.

## Alcance

### In scope

- `Utilities/WhatsAppLinkBuilder.swift` puro (número + mensaje + URL).
- `Services/WhatsAppOpener.swift` que usa `UIApplication.shared.open`.
- Cablear `onWhatsApp` en la lista.
- Alerts de error (número / no instalado).
- Tests del builder (sin UI).

### Out of scope

- WhatsApp Business API, plantillas oficiales, envío automático.
- Usar teléfono alterno si el principal falla (no requerido).
- iMessage / `sms:`.
- Editar el mensaje en UI (hardcoded).

## Tareas

1. `WhatsAppLinkBuilder.digits(from: String) -> String`
2. `WhatsAppLinkBuilder.internationalMexico(_ digits: String) -> String?`
3. `WhatsAppLinkBuilder.message(nombre:fecha:saldoText:) -> String` con el copy exacto.
4. `WhatsAppLinkBuilder.url(phoneDigits:message:) -> URL?`
5. `WhatsAppOpener.open(for alumno:)`:
   - Construir URL.
   - `canOpenURL(whatsapp://)` → si no, callback `.notInstalled`.
   - `open(url)` de `https://wa.me/...`
6. Lista: `onWhatsApp` llama opener; presentar alert según error.
7. Tests:

   | Input | Output |
   |-------|--------|
   | `5512345678` | digits `525512345678`, wa.me host/path esos dígitos |
   | `52 55 1234 5678` | `525512345678` (sin doble 52) |
   | `555` | nil / inválido |
   | mensaje | contiene `Nova Impulsa`, nombre, fecha `dd/MM/yyyy`, saldo |

8. Test de encoding: espacios del mensaje van como `%20` o `+` válidos en query `text`.

## Criterios de aceptación

- En simulador **sin** WhatsApp: tap WhatsApp → alert de instalar; la app no crashea.
- Con número `5512345678`, la URL generada (test) es `https://wa.me/525512345678?text=...` y el texto decodificado coincide con el template.
- El mensaje usa la fecha de próximo pago y el saldo actuales del alumno (si se acaba de editar, el deeplink refleja lo persistido).
- No se usa el teléfono alterno.
- `LSApplicationQueriesSchemes` contiene `whatsapp`; si no, `canOpenURL` siempre da false y el criterio de “no instalado” se rompería en dispositivo real.

## Notas técnicas

- `canOpenURL` en iOS 9+ requiere Info.plist; sin él, iOS loguea y retorna false.
- Abrir `https://wa.me` en simulador puede ir a Safari; es aceptable. El alert de no instalado aplica cuando `whatsapp://` no abre.
- `UIApplication.shared` no está en unit tests: el opener se mockea; los tests unitarios cubren el builder.
- No poner el token ni API de Meta. No hay red propia.
