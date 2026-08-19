# AGENTS.md

Léelo primero. El contrato de ingeniería (arquitectura, Swift, UI, dominio, tests, estilo de cambios) está en [`.cursor/rules/`](.cursor/rules/). No lo dupliques aquí.

## Producto

App nativa de **iPhone** para control escolar y cobranza de **Nova Impulsa**: padrón, pagos, dashboard de cartera y seguimiento. Un operador, offline.

Negocio: [`docs/prds/PRD_Control_Escolar_Nova_Impulsa.md`](docs/prds/PRD_Control_Escolar_Nova_Impulsa.md).

## Stack y carpetas

Un solo target de app (no es monorepo). Swift + SwiftUI + SwiftData conviven ahí; XCTest va en el target de tests.

```
App/           @main, ModelContainer
Models/        @Model Alumno, catálogos
ViewModels/    estado y orquestación
Views/         SwiftUI
Services/      notificaciones, WhatsApp, CSV
Utilities/     PaymentCalculator, formatters
NovaImpulsaTests/
docs/prds/
docs/specs/
```

Quién hace qué en cada carpeta: [`.cursor/rules/`](.cursor/rules/).

## Tests

En Xcode: Product → Test (`⌘U`), esquema **Nova Impulsa**.

```bash
xcodebuild test -scheme "Nova Impulsa" -destination "platform=iOS Simulator,name=iPhone 16"
```

## Flujo de trabajo

Las specs están en [`docs/specs/`](docs/specs/). Se implementan **en orden estricto, una por una, de `01` a `10`**. No adelantes la siguiente hasta cerrar la actual.
