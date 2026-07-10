# Feature Spec: Setup Fundacional del Proyecto

**ID de feature:** `001-setup-fundacional`
**Estado:** Draft — listo para `/plan`
**Depende de:** `constitution.md`, `contexto-negocio.md`

---

## 1. Resumen

Antes de construir cualquier feature de producto (registro de transacciones, Wrapped, etc.), el
proyecto necesita su base técnica: repositorio Flutter con Clean Architecture, 3 entornos
(dev/staging/prod) cada uno con su propio proyecto de Firebase, sistema de theming central
(claro/oscuro, cero valores en duro) y autenticación básica funcionando de punta a punta. Esta
feature es el "esqueleto" sobre el que se construyen todas las demás.

## 2. Por qué (contexto de negocio)

Sin esto, cualquier feature posterior (registro rápido, Wrapped, presupuestos) se construiría
sobre una base inconsistente: sin separación de entornos se arriesga mezclar datos de prueba con
datos reales de usuarios; sin theming central, cada pantalla nueva rompe la regla de "cero colores
en duro" de la constitución; sin Clean Architecture desde el inicio, migrar código ya escrito es
mucho más caro que empezar bien.

## 3. Escenarios de Usuario / Aceptación

> Nota: en esta feature el "usuario" primario es el equipo de desarrollo, no el usuario final —
> es infraestructura. Aun así se listan escenarios verificables.

### Escenario 1 — Cambiar de entorno sin riesgo
**Dado** que un desarrollador corre `flutter run --flavor dev`,
**Cuando** la app inicia,
**Entonces** se conecta exclusivamente al proyecto Firebase `finance-wrapped-dev`, y el nombre del
entorno es visible en un badge discreto dentro de la app (solo en `dev`/`staging`, nunca en `prod`).

### Escenario 2 — Registro/login funcional end-to-end
**Dado** un usuario nuevo,
**Cuando** elige "continuar con Google" (o Apple, o correo),
**Entonces** se crea su documento de usuario en Firestore con los campos base (`uid`, `email`,
`fechaCreacion`, `is_premium_whitelisted: false`), y queda autenticado dentro de la app.

### Escenario 3 — Modo oscuro/claro
**Dado** que el usuario cambia el modo de apariencia del sistema operativo (o lo fuerza manualmente
dentro de ajustes de la app),
**Cuando** vuelve a abrir cualquier pantalla ya construida,
**Entonces** todos los colores, tipografías y espaciados se actualizan automáticamente sin requerir
cambios de código en los widgets de esa pantalla (porque todo viene del `AppTheme` central).

### Escenario 4 — Reglas de seguridad verificadas
**Dado** un usuario autenticado con `uid: A`,
**Cuando** intenta leer o escribir un documento de Firestore perteneciente a `uid: B`,
**Entonces** la operación es rechazada por las Firestore Security Rules (verificado con tests
automatizados sobre el Firestore Emulator, no solo revisión manual).

## 4. Requerimientos Funcionales

- **RF-01**: El repo debe tener la estructura de carpetas `core/` (theme, router, utils
  compartidos, clientes de Firebase) y `features/<nombre>/{data,domain,presentation}/` desde el
  primer commit.
- **RF-02**: Deben existir 3 flavors configurados (`dev`, `staging`, `prod`) ejecutables tanto en
  iOS como Android, cada uno apuntando a su propio proyecto Firebase (ver constitution.md sección 4
  para nombres exactos de proyecto y bundle id).
- **RF-02b**: Deben existir las 3 ramas permanentes del repo (`dev`, `staging`, `master`), con
  protección de rama activada (sin commits directos, solo vía Pull Request) desde el primer commit,
  siguiendo el Git Workflow definido en `constitution.md`.
- **RF-03**: Cada flavor debe tener su propio ícono de app y nombre visible distinguible (ej. "FW
  Dev", "FW Staging", "Finance Wrapped") para evitar confundir instalaciones en el mismo
  dispositivo de prueba.
- **RF-04**: Debe existir un `AppTheme` central (Riverpod provider) con: paleta de colores
  (semántica: `primary`, `background`, `surface`, `success`, `warning`, `error`, `ingreso`,
  `egreso`...), tipografía (escala completa: display/headline/title/body/label), espaciados
  estandarizados, y soporte de modo claro/oscuro vía `ThemeExtension`.
- **RF-05**: Firebase Authentication configurado con proveedores Apple, Google y Correo/Password en
  los 3 proyectos.
- **RF-06**: Al primer login, se debe crear automáticamente el documento de usuario en la colección
  `usuarios/{uid}` con la estructura base definida en `contexto-negocio.md`.
- **RF-07**: Firestore Security Rules desplegadas en los 3 proyectos, garantizando aislamiento
  estricto por `uid`, versionadas en el repo (no editadas solo desde la consola de Firebase).
- **RF-08**: Firebase Analytics y Crashlytics inicializados y funcionando (verificar con un evento
  de prueba visible en el dashboard) en los 3 entornos.
- **RF-09**: Pantalla de recuperación de contraseña y verificación de correo funcionando para el
  proveedor de correo/password.
- **RF-10**: Opción de "Eliminar mi cuenta" en ajustes, que borra el documento del usuario y cierra
  sesión (el borrado en cascada de subcolecciones puede resolverse con una Cloud Function
  disparada por la eliminación del documento raíz — a definir en `plan.md`).
- **RF-11**: Base de datos local `drift` (SQLite) inicializada como fuente de verdad on-device,
  con al menos la tabla de `usuarios` espejada localmente y un mecanismo base de sincronización
  (cola de escrituras pendientes → Firestore) funcionando de punta a punta para esa única entidad,
  sirviendo como plantilla para el resto de features. La estrategia de resolución de conflictos por
  defecto (last-write-wins por timestamp de servidor) definida en `constitution.md` Principio IV
  debe quedar implementada aquí, no reinventada en cada feature futura.

## 5. Fuera de Alcance (explícitamente)

- Ninguna pantalla de producto real (registro de transacciones, Wrapped, presupuestos) — esas son
  features independientes que dependen de esta.
- Integración de RevenueCat (se hace en la feature de Suscripciones/Premium, no aquí — aunque el
  SDK puede quedar instalado sin configurar productos todavía).
- Soporte para Apple Watch / Wear OS (fase posterior, según constitution.md sección 3).
- Internacionalización a inglés (queda como campo `locale` reservado, pero solo copy en español por
  ahora).

## 6. Checklist de Aceptación

- [ ] Los 3 flavors compilan y corren en iOS y Android sin errores.
- [ ] Cada flavor se conecta al proyecto Firebase correcto (verificable revisando el proyecto en
      consola mientras la app corre).
- [ ] Existe al menos una pantalla de ejemplo que consuma `AppTheme` y se vea correctamente en modo
      claro y oscuro.
- [ ] Un usuario puede registrarse, cerrar sesión y volver a iniciar sesión con los 3 métodos de
      auth.
- [ ] Tests de Firestore Security Rules corriendo contra el Emulator, cubriendo al menos el
      escenario 4.
- [ ] Documento de usuario se crea correctamente al primer login, verificado con test de
      integración o inspección manual en Firestore.
- [ ] README del repo documenta cómo levantar cada flavor localmente.
- [ ] La tabla `usuarios` en `drift` refleja el documento creado en Firestore tras el login, y
      sigue funcionando (lectura local) con el dispositivo en modo avión.

## 7. Preguntas Abiertas para `/plan`

- ¿Se usa `flutter_flavorizr` o configuración manual de flavors? (decisión de implementación, no de
  producto — libre para `/plan`).
- ¿La Cloud Function de borrado en cascada se implementa ya en esta feature o se pospone hasta que
  exista contenido real que borrar (transacciones, presupuestos)? Sugerencia: dejar el borrado del
  documento raíz funcionando ahora, y ampliar la Cloud Function conforme se agreguen subcolecciones
  en features futuras.
