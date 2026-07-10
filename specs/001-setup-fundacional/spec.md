# Feature Specification: Setup Fundacional del Proyecto

**Feature Branch**: `001-setup-fundacional`

**Created**: 2026-07-09

**Status**: Draft

**Input**: Draft specification from `.specify/drafts/spec-001-setup-fundacional.md`

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Entorno por Flavor sin Riesgo de Cruce (Priority: P1)

Un desarrollador ejecuta la app con un flavor específico (`dev`, `staging` o `prod`) y la app se conecta exclusivamente al proyecto Firebase correcto para ese entorno. No existe posibilidad de que un flavor acceda a datos de otro entorno.

**Why this priority**: Sin separación de entornos, cualquier desarrollo o prueba pone en riesgo datos reales de usuarios. Es el prerequisito técnico más crítico de toda la app.

**Independent Test**: Compilar y ejecutar cada flavor en iOS y Android; verificar en la consola de Firebase que los eventos de Analytics y las escrituras en Firestore aterrizan en el proyecto correcto. El badge de entorno es visible en `dev`/`staging` pero no en `prod`.

**Acceptance Scenarios**:

1. **Given** un desarrollador ejecuta `flutter run --flavor dev`, **When** la app inicia, **Then** se conecta exclusivamente al proyecto Firebase `finance-wrapped-dev` y muestra un badge "DEV" visible en la UI.
2. **Given** un desarrollador ejecuta `flutter run --flavor staging`, **When** la app inicia, **Then** se conecta exclusivamente al proyecto Firebase `finance-wrapped-staging` y muestra un badge "STAGING".
3. **Given** un desarrollador ejecuta `flutter run --flavor prod`, **When** la app inicia, **Then** se conecta al proyecto Firebase `finance-wrapped-prod` y NO muestra ningún badge de entorno.
4. **Given** los 3 flavors configurados, **When** se compila cada uno en iOS y Android, **Then** los 6 builds compilan sin errores.
5. **Given** los 3 flavors instalados en un mismo dispositivo de prueba, **When** el usuario mira la lista de apps, **Then** cada uno tiene nombre e ícono distinguible ("FW Dev", "FW Staging", "Finance Wrapped").

---

### User Story 2 - Autenticación End-to-End (Priority: P1)

Un usuario nuevo puede registrarse usando Google, Apple o correo/contraseña; al hacerlo, su documento de usuario se crea automáticamente en Firestore con los campos base. Puede cerrar sesión y volver a iniciar sesión. Puede recuperar su contraseña si usa correo/contraseña.

**Why this priority**: Sin autenticación no hay identidad de usuario, y sin identidad no es posible construir ninguna feature de producto (transacciones, Wrapped, etc.). Es co-prioritario con la separación de entornos.

**Independent Test**: Registrarse con cada método de auth, verificar que el documento de usuario existe en Firestore con los campos correctos, cerrar sesión, volver a iniciar sesión, y para correo/password probar el flujo de recuperación de contraseña.

**Acceptance Scenarios**:

1. **Given** un usuario nuevo, **When** elige "Continuar con Google", **Then** se crea su documento en `usuarios/{uid}` con campos `uid`, `email`, `fechaCreacion`, `is_premium_whitelisted: false` y queda autenticado.
2. **Given** un usuario nuevo, **When** elige "Continuar con Apple", **Then** se crea su documento con los mismos campos base y queda autenticado.
3. **Given** un usuario nuevo, **When** se registra con correo y contraseña, **Then** recibe un correo de verificación, se crea su documento y queda autenticado.
4. **Given** un usuario autenticado, **When** cierra sesión y vuelve a iniciar sesión, **Then** accede a su misma cuenta y datos.
5. **Given** un usuario con cuenta de correo/password, **When** solicita recuperación de contraseña, **Then** recibe un correo con link para restablecer y puede iniciar sesión con la nueva contraseña.

---

### User Story 3 - Theming Central Claro/Oscuro (Priority: P2)

Todas las pantallas de la app consumen colores, tipografías y espaciados desde un `AppTheme` central. El usuario puede ver la app en modo claro u oscuro (siguiendo la preferencia del sistema operativo o forzándolo manualmente desde ajustes). Ningún widget usa valores hardcodeados de color, fuente o espaciado.

**Why this priority**: La constitución (Principio III) prohíbe colores hardcodeados. Establecer el theming ahora evita deuda técnica acumulativa en cada feature futura.

**Independent Test**: Construir al menos una pantalla de ejemplo que consuma `AppTheme`. Alternar entre modo claro y oscuro (vía sistema operativo y vía toggle manual en la app). Todos los elementos visuales MUST actualizarse automáticamente sin cambios de código en los widgets.

**Acceptance Scenarios**:

1. **Given** la app ejecutándose con el tema del sistema en modo claro, **When** el usuario cambia el sistema operativo a modo oscuro, **Then** todos los colores, tipografías y espaciados se actualizan automáticamente.
2. **Given** ajustes de la app con opción de tema (Sistema / Claro / Oscuro), **When** el usuario selecciona "Oscuro", **Then** la app cambia a modo oscuro independientemente de la preferencia del sistema.
3. **Given** cualquier pantalla construida en la app, **When** un revisor inspecciona el código del widget, **Then** no encuentra ningún `Color(0x...)`, `TextStyle(...)` con valores literales, ni `EdgeInsets` con valores raw.

---

### User Story 4 - Seguridad de Datos por Usuario (Priority: P2)

Las Firestore Security Rules garantizan aislamiento estricto: un usuario autenticado solo puede leer y escribir sus propios documentos. Esta protección es verificable con tests automatizados contra el Firestore Emulator.

**Why this priority**: Sin reglas de seguridad, cualquier usuario podría acceder a datos financieros de otro. Es un requisito de compliance y de confianza del usuario.

**Independent Test**: Ejecutar tests automatizados contra el Firestore Emulator que verifiquen que `uid: A` no puede leer ni escribir documentos de `uid: B`.

**Acceptance Scenarios**:

1. **Given** un usuario autenticado con `uid: A`, **When** intenta leer un documento perteneciente a `uid: B`, **Then** la operación es rechazada por las Security Rules.
2. **Given** un usuario autenticado con `uid: A`, **When** intenta escribir en un documento perteneciente a `uid: B`, **Then** la operación es rechazada por las Security Rules.
3. **Given** las Security Rules en el repositorio, **When** se despliegan a los 3 proyectos Firebase, **Then** las reglas son idénticas en los 3 entornos.

---

### User Story 5 - Base de Datos Local con Sincronización Base (Priority: P3)

La base de datos local `drift` (SQLite) queda inicializada como fuente de verdad on-device, con al menos la tabla de `usuarios` espejada localmente. Un mecanismo base de sincronización (cola de escrituras pendientes → Firestore) funciona de punta a punta para esa entidad, sirviendo como plantilla para features futuros.

**Why this priority**: Es la base del Principio IV (Offline-First). Implementarla ahora con una sola entidad simple establece el patrón que todas las features futuras seguirán.

**Independent Test**: Crear un usuario, verificar que aparece en la tabla local `drift`. Poner el dispositivo en modo avión, verificar que la lectura local sigue funcionando. Restaurar conectividad y verificar que la sincronización completa los pendientes hacia Firestore.

**Acceptance Scenarios**:

1. **Given** un usuario recién autenticado, **When** se crea su documento en Firestore, **Then** la tabla `usuarios` en `drift` refleja los mismos datos localmente.
2. **Given** el dispositivo en modo avión, **When** el usuario abre la app, **Then** puede ver sus datos de usuario desde la base local sin error.
3. **Given** una escritura pendiente en la cola de sincronización, **When** se restaura la conectividad, **Then** la escritura se envía a Firestore y la cola se vacía.
4. **Given** la estrategia de resolución de conflictos (last-write-wins por timestamp de servidor), **When** ocurre un conflicto entre el dato local y el remoto, **Then** prevalece la escritura con el timestamp más reciente.

---

### User Story 6 - Eliminación de Cuenta (Priority: P3)

El usuario puede eliminar su cuenta desde los ajustes de la app. Esto borra su documento de Firestore, cierra la sesión y lo regresa a la pantalla de login.

**Why this priority**: Requisito de compliance (Principio VII) y buena práctica de privacidad. No es bloqueante para las features de producto, pero debe existir antes de un release público.

**Independent Test**: Crear una cuenta, navegar a ajustes, ejecutar "Eliminar mi cuenta", verificar que el documento del usuario ya no existe en Firestore y que la sesión queda cerrada.

**Acceptance Scenarios**:

1. **Given** un usuario autenticado, **When** selecciona "Eliminar mi cuenta" en ajustes y confirma, **Then** su documento en `usuarios/{uid}` se elimina de Firestore.
2. **Given** la cuenta eliminada, **When** el proceso completa, **Then** la sesión se cierra y el usuario es redirigido a la pantalla de login.
3. **Given** que el usuario tendrá subcolecciones en el futuro, **When** se elimina la cuenta ahora, **Then** el sistema borra el documento raíz y deja preparado el mecanismo para borrado en cascada (Cloud Function) que se expandirá conforme se agreguen subcolecciones.

---

### Edge Cases

- What happens when un usuario intenta registrarse con un correo ya usado por otro método de auth (e.g., Google y luego correo/password con el mismo email)? El sistema debe manejar el linking de cuentas o mostrar un error claro.
- What happens when la conexión de red se pierde durante el proceso de registro? La creación del documento en Firestore debe reintentarse cuando se restaure la conectividad.
- What happens when un usuario elimina su cuenta y luego intenta registrarse de nuevo con el mismo método de auth? Debe poder crear una cuenta nueva sin conflictos.
- How does the system handle Firebase Authentication token expiration? La app debe refrescar tokens automáticamente sin interrumpir la sesión del usuario.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El repositorio MUST tener la estructura de carpetas `lib/core/` (theme, router, utils compartidos, clientes de Firebase) y `lib/features/<nombre>/{data,domain,presentation}/` desde el primer commit funcional.
- **FR-002**: MUST existir 3 flavors configurados (`dev`, `staging`, `prod`) ejecutables tanto en iOS como Android, cada uno apuntando a su propio proyecto Firebase (`finance-wrapped-dev`, `finance-wrapped-staging`, `finance-wrapped-prod`).
- **FR-003**: MUST existir las 3 ramas permanentes del repo (`dev`, `staging`, `master`) con protección de rama activada (sin commits directos, solo vía PR), siguiendo el Git Workflow definido en la constitución.
- **FR-004**: Cada flavor MUST tener su propio ícono de app y nombre visible distinguible ("FW Dev", "FW Staging", "Finance Wrapped") para evitar confusión en el mismo dispositivo.
- **FR-005**: MUST existir un `AppTheme` central (Riverpod provider) con paleta de colores semántica (`primary`, `background`, `surface`, `success`, `warning`, `error`, `ingreso`, `egreso`), tipografía completa (display/headline/title/body/label), espaciados estandarizados, y soporte de modo claro/oscuro vía `ThemeExtension`.
- **FR-006**: Firebase Authentication MUST estar configurado con proveedores Apple, Google y Correo/Password en los 3 proyectos Firebase.
- **FR-007**: Al primer login, MUST crearse automáticamente el documento de usuario en `usuarios/{uid}` con campos base (`uid`, `email`, `fechaCreacion`, `is_premium_whitelisted: false`).
- **FR-008**: Firestore Security Rules MUST estar desplegadas en los 3 proyectos, garantizando aislamiento estricto por `uid`, versionadas en el repositorio.
- **FR-009**: Firebase Analytics y Crashlytics MUST estar inicializados y funcionando en los 3 entornos (verificable con un evento de prueba en el dashboard).
- **FR-010**: MUST existir pantalla de recuperación de contraseña y verificación de correo para el proveedor correo/password.
- **FR-011**: MUST existir opción de "Eliminar mi cuenta" en ajustes que borra el documento del usuario y cierra sesión.
- **FR-012**: Base de datos local `drift` (SQLite) MUST estar inicializada con al menos la tabla `usuarios` espejada localmente y un mecanismo base de sincronización (cola de escrituras pendientes → Firestore) funcionando end-to-end para esa entidad.
- **FR-013**: La estrategia de resolución de conflictos por defecto (last-write-wins por timestamp de servidor) MUST quedar implementada en esta feature como plantilla para features futuros.

### Key Entities *(include if feature involves data)*

- **Usuario**: Representa la identidad del usuario en la app. Campos base: `uid` (string, PK), `email` (string), `fechaCreacion` (timestamp), `is_premium_whitelisted` (boolean, default false). Existe tanto en Firestore (`usuarios/{uid}`) como en la tabla local `drift`.
- **SyncQueue**: Cola de escrituras pendientes para sincronización offline → online. Cada entrada contiene: referencia a la entidad, operación (create/update/delete), payload, timestamp de creación, estado (pending/synced/failed), número de reintentos.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Los 3 flavors compilan y ejecutan sin errores en iOS y Android (6 builds exitosos).
- **SC-002**: Un usuario nuevo puede completar el registro con cualquiera de los 3 métodos de auth en menos de 60 segundos (excluyendo tiempo de verificación de correo).
- **SC-003**: El 100% de los tests automatizados de Firestore Security Rules pasan contra el Emulator, cubriendo al menos los escenarios de aislamiento por `uid`.
- **SC-004**: La app funciona correctamente en modo avión: el usuario puede ver sus datos locales sin error ni spinner bloqueante.
- **SC-005**: Al alternar entre modo claro y oscuro, todos los elementos visuales se actualizan en menos de 1 segundo sin requerir reinicio de la app.
- **SC-006**: El documento de usuario se crea correctamente al primer login, verificable con test de integración.
- **SC-007**: La tabla `usuarios` en `drift` refleja el documento de Firestore tras el login, y sigue accesible con el dispositivo offline.

## Assumptions

- Los 3 proyectos Firebase (`finance-wrapped-dev`, `finance-wrapped-staging`, `finance-wrapped-prod`) ya están creados en la consola de Firebase con los bundle IDs correctos.
- El desarrollador tiene acceso de administrador a los 3 proyectos Firebase.
- Las cuentas de Apple Developer y Google Play Developer están configuradas para soportar Sign in with Apple y Google Sign-In respectivamente.
- El borrado en cascada de subcolecciones al eliminar cuenta se resolverá con una Cloud Function que se expandirá conforme se agreguen features con subcolecciones; en esta feature solo se borra el documento raíz.
- RevenueCat SDK puede quedar instalado como dependencia sin configurar productos todavía.
- Solo se soporta español como idioma en esta fase; el campo `locale` queda reservado para internacionalización futura.
- No se incluye ninguna pantalla de producto (registro de transacciones, Wrapped, presupuestos) en esta feature.

## Out of Scope

- Pantallas de producto (registro de transacciones, Wrapped, presupuestos, Fondo de Ahorro) — son features independientes que dependen de esta.
- Integración completa de RevenueCat con productos configurados — se hace en la feature de Suscripciones/Premium.
- Soporte para Apple Watch / Wear OS — fase posterior.
- Internacionalización a inglés — solo español por ahora.
