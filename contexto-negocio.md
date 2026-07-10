# Contexto de Negocio — Finance Wrapped

> Documento de referencia (no es un spec de Spec Kit). Contiene las decisiones de producto
> que los distintos `spec.md` de features deben usar como fuente de verdad para no
> contradecirse entre sí. Se actualiza conforme el producto evolucione.

---

## 1. Detección de Ingreso vs. Egreso

No se le pide al usuario elegir explícitamente "ingreso" o "egreso" en el flujo rápido de
registro — rompería el objetivo de 3 segundos. En su lugar:

- **Cada categoría tiene un `tipo` fijo**: `ingreso` o `egreso`. Ej: "Nómina", "Freelance",
  "Reembolso" → `ingreso`. "Comida", "Transporte", "Servicios" → `egreso`.
- Al crear una categoría personalizada, el usuario **sí** elige su tipo una sola vez (al crearla,
  no en cada registro).
- Las píldoras inteligentes (categorías recientes) muestran una categoría a la vez con su tipo ya
  resuelto — el monto se guarda como positivo internamente y el campo `tipo` en el documento de
  transacción determina si suma o resta en los cálculos.
- Excepción: la categoría especial **"Fondo de Ahorro"** puede recibir tanto aportaciones
  (`egreso` desde la cuenta corriente hacia el fondo) como retiros (`ingreso` desde el fondo hacia
  gasto corriente) — ver sección 2.

## 2. Fondo de Emergencia / Ahorro

- El usuario registra una transacción con categoría **"Fondo de Ahorro"** como si fuera un egreso
  normal (ej. "Aporté $500 al fondo"). Esto incrementa un contador `fondoAhorro.saldo` en el
  perfil del usuario (o subcolección `metas/fondoEmergencia`).
- Cuando el usuario necesita usar esos ahorros, la categoría "Fondo de Ahorro" aparece como
  **método de origen** disponible al registrar un gasto (ej. pagar una reparación del coche desde
  el fondo). Esa transacción se marca internamente con `origen: fondoAhorro` además de su
  categoría real de gasto.
- En el Wrapped (mensual o anual), cualquier transacción con `origen: fondoAhorro` se agrupa y
  presenta como **"Uso de Fondo de Emergencia"** — nunca se penaliza ni se muestra como un mal
  hábito; el copy debe felicitar al usuario por haber estado preparado.
- El saldo del fondo es visible en un widget simple dentro de la app (no forma parte del registro
  rápido de 3 segundos, vive en una pantalla de "Metas" o similar — a definir en spec de esa
  feature).

## 3. Retención de Historial (Free vs. Premium)

- Usuarios **Free**: solo pueden ver/consultar transacciones del **mes calendario en curso**. Al
  cambiar de mes, el mes anterior queda oculto en la UI (los datos NO se borran de Firestore, solo
  se restringe el acceso vía la capa de dominio/reglas de UI — importante por si el usuario luego
  se vuelve Premium y quiere recuperar visibilidad).
- Usuarios **Premium**: acceso a historial completo, pero **únicamente de los meses en los que la
  cuenta tuvo Premium activo**. Si el usuario paga Premium en agosto, puede ver agosto en adelante
  mientras esté activo; los meses previos a su primera suscripción permanecen bloqueados aunque
  haya datos (se muestran como "Desbloquea tu historial completo" con upsell).
- Regla derivada: cada transacción, al crearse, se marca con un flag `capturadaEnPremium: bool`
  para poder resolver este límite de forma eficiente sin recalcular fechas de suscripción cada vez.

## 4. Personalidades Financieras (Wrapped Mensual)

Set inicial de 10 arquetipos. Se elige por reglas (no ML) basadas en métricas del mes: categoría
dominante, frecuencia de registro, variación vs. mes anterior, uso de presupuestos, día de
descontrol, etc. El detalle exacto de las reglas de asignación se define en el spec del Motor
Wrapped, no aquí — esta es solo la lista de personajes a diseñar (arte/copy):

1. **El Estratega** — presupuestó y se mantuvo dentro del límite en casi todas las categorías.
2. **El Espontáneo** — gasto muy distribuido, sin categoría dominante clara, muchas compras pequeñas.
3. **El Fiel a la Comida** — categoría "Comida/Restaurantes" domina el mes por mucho margen.
4. **El Cazador de Ofertas** — muchas transacciones pequeñas en la misma categoría (compras frecuentes de bajo monto).
5. **El Previsor** — aportó al Fondo de Ahorro este mes, sin haber retirado.
6. **El Bombero** — tuvo que usar el Fondo de Emergencia este mes (tono celebratorio, no de culpa).
7. **El Fantasma** — registró menos de 10 transacciones (activa el modo minimalista, ver sección 5).
8. **El Ordenado** — usó consistentemente notas/metadatos opcionales (método de pago, notas) en casi todos sus registros.
9. **El de Fin de Quincena** — patrón de gasto claramente concentrado los días posteriores a la fecha de nómina detectada.
10. **El Sorpresa** — tuvo un solo gasto atípico muy por encima de su promedio habitual ("Día de Descontrol" con desviación extrema vs. promedio).

> Nota: este set es punto de partida; se pueden agregar más conforme se validen datos reales de usuarios.

## 5. Modo "Pocos Datos" (< 10 transacciones en el mes)

- Se reemplaza el Wrapped narrativo normal por una pantalla directa, minimalista y con tono
  humorístico (copy a cargo de una feature de contenido, no de este documento).
- Debe mostrar como mínimo: el único gasto relevante del mes (el de mayor monto) y un tip corto
  para fomentar el hábito de registro diario.
- Este modo aplica igual a usuarios Free y Premium — no es una limitación de plan, es una
  limitación de datos disponibles.

## 6. "Día de Descontrol"

- Definición operativa: el día del mes calendario cuyo gasto total tuvo la **mayor desviación
  respecto al promedio diario de gasto del usuario en ese mes** (no simplemente el día de mayor
  gasto absoluto, para evitar que un solo gasto fijo grande — como renta — siempre "gane").

## 7. Exportación / Compartible del Wrapped

- Formato: imagen estática (PNG) tipo story vertical (1080x1920), generada **client-side** con
  Flutter (`RepaintBoundary` + captura de widget a imagen) — es la opción más barata: no requiere
  infraestructura de renderizado server-side ni costos de Cloud Functions por imagen generada.
- Todos los usuarios (Free y Premium) pueden exportar y compartir su Wrapped.
- La marca de agua de la app aparece siempre por default; **solo usuarios Premium** pueden
  desactivarla antes de exportar.

## 8. Presupuestos

- Totalmente opcionales; el usuario los crea manualmente por categoría y periodo (el usuario
  define el monto acorde a sus propios pagos/ingresos, sin sugerencias automáticas en esta fase).
- Actualmente solo se reflejan como parte del análisis del Wrapped mensual (no hay notificaciones
  push de "estás por pasarte del presupuesto" en el alcance actual — queda como posible feature
  futura, no incluida en el MVP).

## 9. Cuentas Compartidas (Parejas/Familia) — Fuera de alcance del MVP

- El modelo de datos actual es **individual** (`userId` como dueño único de cada documento).
- Se debe diseñar el esquema de Firestore dejando espacio para una futura migración a "espacios
  compartidos" (ej. un `householdId` opcional a futuro), pero **no se construye la feature ahora**.
  Ningún spec del MVP debe implementar UI ni lógica de compartición de datos entre usuarios.

## 10. Estudio de Mercado / Precio (referencia)

| Competidor | Mercado | Modelo | Precio |
|---|---|---|---|
| YNAB | EEUU | Solo pago, sin tier gratis | $109/año o $14.99/mes |
| Rocket Money | EEUU | Freemium, "paga lo que quieras" | $7–14/mes |
| Fintonic | España/México | Freemium, conecta bancos | Gratis / ~$99 MXN/mes Premium |

Decisión: **$49 MXN/mes** o **$399 MXN/año**, posicionado por debajo de Fintonic (que además
conecta bancos automáticamente, un valor mayor) pero con una app más ligera y enfocada en el
hábito de registro + la experiencia narrativa como diferenciador, no en agregación bancaria.

## 11. Pasarela de Pagos

**RevenueCat** sobre `purchases_flutter`. Motivo: abstrae StoreKit + Google Play Billing en una
sola integración, incluye validación de recibos, manejo de renovaciones/cancelaciones/reembolsos y
"restore purchases" out-of-the-box — evita construir y mantener esa infraestructura a mano.
El flag `is_premium_whitelisted` en Firestore se gestiona manualmente (sin panel de admin en el
MVP) y se combina con el estado de RevenueCat para resolver el entitlement final.

## 12. Cuentas Individuales, Autenticación

- Firebase Auth con Apple / Google / Correo-contraseña.
- Debe incluir: recuperación de contraseña, verificación de correo (para el método
  correo/contraseña), y **borrado de cuenta autoservicio** desde ajustes (requisito de las tiendas
  y de la LFPDPPP).
