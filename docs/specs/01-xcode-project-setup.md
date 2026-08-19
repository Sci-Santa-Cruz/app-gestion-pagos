# Spec 01 — Xcode project setup

## Objetivo

Crear el proyecto nativo iOS de Nova Impulsa con Swift, SwiftUI, SwiftData, arquitectura MVVM, destino solo iPhone e iOS 17+, listo para las specs siguientes.

## Contexto y dependencias

- **Fuente de verdad:** `docs/prds/PRD_Control_Escolar_Nova_Impulsa.md`
- **Dependencias:** ninguna. Esta es la primera spec.
- **Producto:** app interna de control escolar y cobranza, un operador, 100% offline, sin login ni backend.
- **RNF aplicables:** RNF-001 (solo iPhone), RNF-002 (Swift + SwiftUI + SwiftData, iOS 17+), RNF-003 (sin backend ni sync), RNF-004 (offline), RNF-005 (español MX).

Si el repo aún no tiene `.xcodeproj`, créalo. Si ya existe un proyecto, ajústalo a este contrato en lugar de duplicarlo.

## Alcance

### In scope

- Proyecto Xcode (iOS App) con SwiftUI lifecycle.
- Deployment target **iOS 17.0**.
- Destino **iPhone only** (`TARGETED_DEVICE_FAMILY = 1`). Sin iPad.
- SwiftData: `ModelContainer` en el `@main` App, almacenamiento local, **sin CloudKit**.
- Carpetas MVVM vacías pero compilables.
- Target de unit tests.
- UI en español; locale de formato `es_MX`.
- Nombre visible: **Nova Impulsa**.
- Claves de Info.plist que más adelante usarán notificaciones y WhatsApp.

### Out of scope

- Modelos SwiftData de alumno (spec 02).
- Pantallas de dashboard, formularios, CSV, WhatsApp o notificaciones funcionales.
- Autenticación, backend, CocoaPods/SPM de terceros (no se necesitan).
- Soporte iPad, widgets, App Clips.

## Tareas

1. Crear (o ajustar) la app iOS:
   - Product Name: `Nova Impulsa`
   - Bundle Identifier: `com.novaimpulsa.controlescolar`
   - Interface: SwiftUI
   - Language: Swift
   - Storage: None en el wizard; el container SwiftData se cablea a mano.
2. Configurar el target:
   - iOS 17.0+
   - Devices: iPhone
   - Orientation: Portrait (Landscape opcional; no es requisito).
   - Display Name: `Nova Impulsa`
3. Crear grupos/carpetas:
   - `App/` — `NovaImpulsaApp.swift`
   - `Models/` — placeholder `.gitkeep` o archivo vacío no es necesario si el grupo existe en Xcode
   - `ViewModels/`
   - `Views/`
   - `Services/`
   - `Utilities/`
   - `Resources/` — `Localizable.xcstrings` o `Localizable.strings` en `es`
4. Implementar `NovaImpulsaApp`:
   - `@main` con `WindowGroup` y una vista raíz temporal `RootPlaceholderView` que muestre el texto «Nova Impulsa» centrado.
   - `.modelContainer(for: [], inMemory: false)` por ahora con arreglo vacío de modelos (spec 02 lo reemplaza). Si SwiftData exige un modelo, crear `PlaceholderModel` mínimo **solo si no compila** el container vacío; márcalo `TODO(spec-02)` para borrarlo.
5. Crear target `NovaImpulsaTests` (XCTest) con un test dummy `testProjectCompiles` que haga `XCTAssertTrue(true)` para validar el pipeline.
6. Info.plist / target Info:
   - `NSUserNotificationsUsageDescription` / texto de permiso: «Nova Impulsa usa avisos locales para recordarte cobros pendientes.» (la lógica llega en spec 08; la clave debe existir ya).
   - `LSApplicationQueriesSchemes` = `whatsapp` (el deeplink llega en spec 09).
   - `CFBundleAllowMixedLocalizations` no es necesario.
7. Localización: Development Language = Spanish. Formateadores de dinero/fecha se centralizarán en spec 02/03; aquí basta el idioma del proyecto.
8. Verificar que el esquema corre en simulador iPhone (p. ej. iPhone 16) y que no hay capability de iCloud.

## Criterios de aceptación

- El esquema `Nova Impulsa` compila sin errores en Xcode 15+ / iOS 17 SDK.
- Al lanzar en simulador iPhone se ve la pantalla placeholder con «Nova Impulsa».
- El destino no incluye iPad (no hay destino iPad en el target).
- `IPHONEOS_DEPLOYMENT_TARGET` es `17.0`.
- No hay `com.apple.developer.icloud-container-identifiers` ni CloudKit capability.
- Existe target de tests y `NovaImpulsaTests` pasa.
- Info.plist (o la pestaña Info del target) contiene la descripción de notificaciones y el scheme `whatsapp`.
- La estructura de carpetas App / Models / ViewModels / Views / Services / Utilities está en el proyecto.

## Notas técnicas

- Arquitectura **MVVM**: vistas SwiftUI, estado en `ObservableObject` / `@Observable` ViewModels, persistencia solo vía SwiftData. No poner lógica de negocio en `App`.
- Preferir `@Observable` (iOS 17) para ViewModels nuevos.
- Dinero se persistirá como `Decimal` (spec 02). No usar `Float`/`Double` para montos.
- No agregar dependencias externas.
- Si se genera `PlaceholderModel`, debe vivir en `Models/` y eliminarse en spec 02 al registrar `Alumno`.
- El agente no debe crear backend, Firebase ni Auth.
