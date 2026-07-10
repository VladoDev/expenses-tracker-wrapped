# Tech Stack — Finance Wrapped

> Este documento detalla el **cómo** técnico de cada decisión ya fijada en `constitution.md`
> (que define el **qué es innegociable**). Sirve como referencia para `/plan` en cada feature:
> cuando un plan necesite un paquete, primero se busca aquí antes de introducir uno nuevo.
> Las versiones exactas de cada paquete deben verificarse en pub.dev al momento de correr
> `flutter pub add`, ya que este documento fija la **elección**, no el pin exacto de versión.

---

## 1. Núcleo

| Elemento | Elección | Notas |
|---|---|---|
| Lenguaje | Dart | Última estable (Dart 3.10+ a la fecha de este documento) |
| Framework | Flutter | Última estable del canal `stable` (Flutter 3.44+ a la fecha de este documento) |
| Gestión de versión Flutter/Dart | `fvm` (Flutter Version Management) | Evita el clásico "en mi máquina funciona" al fijar la misma versión de Flutter para todo el equipo, versionado en `.fvmrc` dentro del repo |

## 2. Arquitectura & Estado

| Necesidad | Paquete | Notas |
|---|---|---|
| State management | `flutter_riverpod` | Fijado en constitution.md — único approach permitido |
| Codegen de providers | `riverpod_annotation` + `riverpod_generator` | Genera providers desde anotaciones `@riverpod`, reduce boilerplate y errores de tipos |
| Modelos inmutables (entities, DTOs, estados de UI) | `freezed` | Usa el modo "mixed mode" (Freezed 3.x) — genera `copyWith`, `==`, unions selladas para estados (`AsyncValue`-like) |
| Serialización JSON (para DTOs de Firestore) | `json_serializable` | Va de la mano con `freezed` para los modelos de la capa `data` |
| Codegen runner | `build_runner` | Corre `dart run build_runner build --delete-conflicting-outputs` en CI antes de tests/build |

**Regla de capas reforzada por tooling**: se agrega `import_lint` (o regla equivalente de
`custom_lint`) configurado para que un archivo bajo `domain/` no pueda importar nada de
`package:flutter/*`, `package:cloud_firestore/*` ni `package:drift/*`. Esto convierte el
Principio I de la constitution (Clean Architecture) en algo verificado por CI, no solo por
revisión humana.

## 3. Persistencia Local (fuente de verdad — Principio IV)

| Necesidad | Paquete | Notas |
|---|---|---|
| Base de datos local | `drift` | ORM tipado sobre SQLite; fuente de verdad on-device |
| Motor SQLite nativo | `sqlite3_flutter_libs` | Requerido por `drift` en iOS/Android |
| Migraciones | `drift`'s schema migration API | Versionadas dentro del repo; cada cambio de esquema requiere un test de migración |
| Ruta de almacenamiento | `path_provider` | Resuelve el directorio de documentos de la app para el archivo `.sqlite` |
| Cola de sincronización (local → Firestore) | Tabla propia `sync_queue` dentro de `drift` (no un paquete externo) | Cada fila representa una operación pendiente (`insert`/`update`/`delete` con payload); un `Worker` en `core/sync` la procesa cuando hay conectividad. Se prefiere esto sobre un paquete de sync genérico para mantener control total sobre la estrategia de conflictos definida en la constitution |
| Detección de conectividad | `connectivity_plus` | Dispara el procesamiento de `sync_queue` al recuperar señal |

## 4. Backend / Firebase

| Necesidad | Paquete | Notas |
|---|---|---|
| Núcleo Firebase | `firebase_core` | Inicialización multi-flavor (`DefaultFirebaseOptions` generado por flavor con FlutterFire CLI) |
| Autenticación | `firebase_auth` | Base para los 3 proveedores |
| Google Sign-In | `google_sign_in` | Requiere configuración de OAuth client por flavor |
| Apple Sign-In | `sign_in_with_apple` | Obligatorio en iOS si se ofrece login social (requisito de Apple) |
| Sync remoto / backup | `cloud_firestore` | Solo accedido desde la capa `data`, nunca desde `presentation` (Principio IV) |
| Analítica | `firebase_analytics` | Solo eventos agregados/anónimos (Principio VII) |
| Crash reporting | `firebase_crashlytics` | Activado en los 3 flavors, con `flutter_error_handler` capturando errores no atrapados |
| Cloud Functions (borrado en cascada, tareas server-side puntuales) | `cloud_functions` (cliente) + funciones en TypeScript en `/functions` | Solo se usa donde de verdad se necesita lógica server-side; la mayoría de la lógica vive en el cliente por diseño offline-first |

## 5. Pagos / Suscripciones

| Necesidad | Paquete | Notas |
|---|---|---|
| Suscripciones in-app | `purchases_flutter` (RevenueCat) | Fijado en constitution.md; `isPremium = revenueCat.isActive OR firestore.is_premium_whitelisted` |

## 6. Navegación

| Necesidad | Paquete | Notas |
|---|---|---|
| Router | `go_router` | Navegación declarativa, deep-linking (útil para notificaciones futuras y para compartir un Wrapped específico), se integra bien con Riverpod vía `riverpod`+`go_router` refresh listenable |

## 7. UI, Theming y Narrativa Visual (el "Wrapped")

| Necesidad | Paquete | Notas |
|---|---|---|
| Theming central | Flutter `ThemeExtension` (sin paquete externo) | Ya cubierto por Principio III de la constitution; no se introduce un paquete de theming de terceros para no perder control fino |
| Tipografía | Fuentes empaquetadas localmente en `assets/fonts/` (NO `google_fonts` en runtime) | Evita depender de descarga en tiempo de ejecución (falla en modo avión) y garantiza consistencia visual offline desde el primer frame |
| Animaciones de las "Stories" del Wrapped | Flutter `AnimationController` / `Hero` / `PageView` nativos como base | Suficiente para transiciones de story (fade, slide, scale) sin dependencia extra |
| Ilustraciones/avatares animados de las 10 Personalidades Financieras | `rive` | Permite a diseño exportar animaciones vectoriales interactivas y livianas (mucho más ligero que video/Lottie para este caso), controlables por estado desde Dart (ej. reaccionar a datos del usuario) |
| Gráficas simples (barras de categorías, comparativos) | `fl_chart` | Solo si el diseño de una pantalla específica lo requiere; se prefiere `CustomPainter` a mano para las piezas más "signature" del Wrapped donde el diseño personalizado importa más que la conveniencia de una librería genérica |

## 8. Exportar / Compartir el Wrapped

| Necesidad | Paquete | Notas |
|---|---|---|
| Captura de widget a imagen | `RepaintBoundary` + `dart:ui` (nativo de Flutter, sin paquete extra) | Renderiza el story actual a un `png` en memoria — enfoque más barato posible, sin backend de renderizado (ver `contexto-negocio.md` §7) |
| Compartir la imagen generada | `share_plus` | Abre el sheet nativo de compartir (Instagram Stories, WhatsApp, etc.) |
| Guardar en galería (opcional) | `gal` | Alternativa moderna y mantenida a `image_gallery_saver` (que está deprecado) |

## 9. Voz (fase Smartwatch — no MVP)

| Necesidad | Paquete / Enfoque | Notas |
|---|---|---|
| Apple Watch | Código nativo Swift/WatchOS + `WatchConnectivity` | No es Flutter; app companion independiente que sincroniza con el device principal |
| Wear OS | Kotlin nativo o Flutter con `wear` plugin, a decidir en el spec de esa fase | Reservado — no se decide en este documento para no comprometer una elección que depende de cómo evolucione el soporte de Wear en Flutter para esa fecha |

## 10. Testing

| Necesidad | Paquete | Notas |
|---|---|---|
| Tests unitarios / widget | `flutter_test` (SDK) | Base |
| Mocking | `mocktail` | Preferido sobre `mockito` — no requiere codegen, sintaxis más simple con null-safety |
| Firestore fake para tests de repos | `fake_cloud_firestore` | Permite testear repositorios de la capa `data` sin backend real ni emulator para tests unitarios rápidos |
| Auth fake para tests | `firebase_auth_mocks` | Igual que arriba, para flujos que dependen de `currentUser` |
| Tests de Firestore Security Rules | `@firebase/rules-unit-testing` (Node, corre vía Firebase Emulator Suite) | Vive en `/firestore-tests`, no en el paquete Flutter — es JS/TS porque así lo requiere el Emulator Suite |
| Tests de integración end-to-end | `integration_test` (SDK) | Cubre los "critical user journeys" que exige el Principio V (registrar transacción, generar Wrapped) |
| Golden tests (consistencia visual del theme) | `golden_toolkit` | Opcional pero recomendado dado que el Principio III depende de consistencia visual estricta — un cambio accidental de color/tipografía se detecta con un diff de imagen en CI |

## 11. Linting & Análisis Estático

| Necesidad | Paquete | Notas |
|---|---|---|
| Reglas base | `flutter_lints` | Punto de partida oficial de Google |
| Reglas adicionales estrictas | `very_good_analysis` (reemplaza/extiende a `flutter_lints`) | Más estricto por defecto (line length, prefer const, etc.), reduce discusiones de estilo en code review |
| Enforcement de límites de arquitectura | `custom_lint` + `import_lint` (ver sección 2) | Falla el build si `domain/` importa de `data/` o de Flutter |

## 12. CI/CD

| Necesidad | Herramienta | Notas |
|---|---|---|
| Pipeline | **GitHub Actions** | Se elige sobre Codemagic para no sumar otra suscripción de pago dado el estado inicial del proyecto; el repo ya vive en GitHub por el uso de Spec Kit |
| Automatización de firma y publicación a tiendas | `fastlane` (lanes separados por flavor) | Estándar de facto para iOS + Android, se integra directo en workflows de GitHub Actions |
| Distribución a testers (staging) | Firebase App Distribution | Encaja natural con que `staging` ya es un proyecto Firebase propio |
| Workflows mínimos sugeridos | `pr-checks.yml` (analyze + test + build_runner en cada PR a `dev`), `deploy-staging.yml` (en merge a `staging`, build + Firebase App Distribution), `deploy-prod.yml` (en merge a `master`, build + submit a stores vía fastlane) | Alineado 1:1 con el Git Workflow de la constitution |

---

## 13. Explícitamente Fuera de Alcance (para no sobre-ingenierizar el MVP)

- **GraphQL / API REST propia**: no hay backend propio más allá de Firebase + eventuales Cloud
  Functions puntuales.
- **Paquete de sync genérico de terceros** (ej. PowerSync, Supabase sync engines): se construye
  una cola de sync propia y simple sobre `drift`, suficiente para el volumen de datos de un
  usuario individual registrando transacciones manuales.
- **CMS o backend de contenido remoto** para el copy/tips del modo "Pocos Datos": el copy vive
  como texto embebido versionado en el repo (localización futura si aplica), no se justifica un
  CMS remoto en el MVP.
- **Librería de gráficos avanzada tipo `syncfusion_flutter_charts`**: de pago/licenciada y con más
  capacidad de la que el Wrapped necesita; se prefiere `fl_chart` (gratis) + `CustomPainter` a mano
  para las piezas más "signature".

---

## 14. Tabla resumen (pubspec.yaml — dependencias principales)

```yaml
dependencies:
  flutter_riverpod:
  riverpod_annotation:
  freezed_annotation:
  json_annotation:
  drift:
  sqlite3_flutter_libs:
  path_provider:
  connectivity_plus:
  firebase_core:
  firebase_auth:
  google_sign_in:
  sign_in_with_apple:
  cloud_firestore:
  firebase_analytics:
  firebase_crashlytics:
  cloud_functions:
  purchases_flutter:
  go_router:
  rive:
  fl_chart:
  share_plus:
  gal:

dev_dependencies:
  build_runner:
  riverpod_generator:
  freezed:
  json_serializable:
  drift_dev:
  flutter_lints: # o very_good_analysis
  custom_lint:
  import_lint:
  mocktail:
  fake_cloud_firestore:
  firebase_auth_mocks:
  integration_test:
  golden_toolkit:
```

> Todas las versiones exactas se resuelven al correr `flutter pub add <paquete>` al iniciar
> `001-setup-fundacional` — este documento fija la elección, pub.dev fija el número.
