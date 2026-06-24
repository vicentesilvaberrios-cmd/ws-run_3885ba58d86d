# Resultado — Momo (FIX incremental): Email de confirmación al reservar

## Alcance del run

Se aplicó un **fix incremental** sobre el proyecto existente (Next.js App Router +
TypeScript + Supabase) para añadir **únicamente** el envío de un **email de
confirmación** en el momento de crear una reserva, tanto desde la página pública del
cliente como desde el panel (cita manual). No se reescribió la base existente: se
respetó el esquema, el multi-tenant por `org_id`, el uso de RPC del lado servidor y el
estilo de formato de fecha/hora de la app. **No** se agregaron recordatorios
programados ni cron.

Stack real verificado en `package.json`: `next ^14.2.35`, `react ^18.3.1`,
`@supabase/ssr`, `@supabase/supabase-js`, TypeScript. El email se envía vía Resend
usando `fetch` al endpoint HTTP (no se agregó dependencia nueva).

---

## Qué se construyó por módulo

### Módulo: Envío de email (nuevo helper)
**Archivo:** `lib/email.ts`

- Exporta `sendConfirmationEmail({ to, businessName, serviceName, startsAt })`.
- Lee la clave **solo del servidor**: `process.env.RESEND_API_KEY` y
  `process.env.RESEND_FROM`. No se usa prefijo `NEXT_PUBLIC` para la API key.
- **Degradación con gracia:** si `RESEND_API_KEY` no está configurada, registra un
  `console.warn` y retorna sin lanzar error (la reserva no se rompe).
- Hace `POST https://api.resend.com/emails` con `Authorization: Bearer <apiKey>`.
- Si Resend responde con error o la llamada falla, lo captura y registra con
  `console.warn` (no propaga la excepción).
- El cuerpo HTML incluye: **nombre del negocio**, **servicio**, **fecha** y **hora**.
- La fecha/hora se formatean con los helpers existentes `formatTime` y `formatDate`
  de `lib/format.ts`.

### Helper de formato (existente, reutilizado)
**Archivo:** `lib/format.ts`

- `formatTime` produce la hora con `timeZone: 'America/Santiago'`, `hour12: false` y
  `hourCycle: 'h23'` (hora de Chile, formato 24h), consistente con el resto de la app.
- `formatDate` formatea la fecha en texto (día de semana, día, mes).

### Reserva desde la página pública del cliente
**Archivo:** `app/api/public/[slug]/appointments/route.ts` (route handler `POST`)

- Crea la reserva vía RPC `create_appointment` (respeta la lógica/constraints del
  esquema, incluido el control de solapamiento que devuelve 409 cuando el horario ya
  no está disponible).
- Tras crear la reserva con éxito, obtiene `organizations.name` (por `slug`) y
  `services.name` (por `serviceId`) y llama a `sendConfirmationEmail`.
- El envío está envuelto en `try/catch` con `console.warn`: si falla el email, la
  reserva ya creada se conserva y responde `201`.

### Reserva manual desde el panel
**Archivo:** `app/api/appointments/route.ts` (route handler `POST`)

- Requiere organización autenticada (`getCurrentOrg` de `lib/org.ts`).
- Crea la cita vía RPC `owner_create_appointment`.
- Tras crear con éxito, obtiene el nombre del servicio y usa `org.name` del contexto
  para llamar a `sendConfirmationEmail`.
- Devuelve `{ id, email_sent }`; `email_sent` se marca `false` si el envío falla, sin
  romper la creación de la reserva (degradación con gracia).
- El `GET` del mismo archivo (listado de citas por fecha) ya existía y no se modificó
  en su comportamiento.

---

## Archivos relevantes en este run

Nuevos / modificados para el fix:
- `lib/email.ts` — helper de envío de confirmación vía Resend (servidor).
- `app/api/public/[slug]/appointments/route.ts` — POST público + envío de email.
- `app/api/appointments/route.ts` — POST panel + envío de email (`email_sent`).

Existentes reutilizados:
- `lib/format.ts` — `formatTime` / `formatDate` (hora de Chile, 24h).
- `lib/org.ts` — `getCurrentOrg` (multi-tenant por `org_id`).
- `lib/supabase/server.ts` — cliente Supabase de servidor.
- `supabase/migrations/0001_init.sql` … `0004_clients.sql` — esquema (no modificado).

---

## Cómo correrlo

1. Instalar dependencias: `npm install`.
2. Variables de entorno requeridas para el email:
   - `RESEND_API_KEY` (solo servidor, **sin** `NEXT_PUBLIC`).
   - `RESEND_FROM` (opcional; si falta, usa `Confirmaciones <no-reply@resend.dev>`).
   - Además las ya existentes de Supabase del proyecto.
3. Desarrollo: `npm run dev`.
4. Build/producción: `npm run build` y `npm start`.

Si `RESEND_API_KEY` no está definida, la app funciona igual: las reservas se crean y
solo se registra en log que el email no se envió.

---

## Criterios de aceptación CUBIERTOS

- ✅ Email de confirmación al crear reserva desde la página pública del cliente.
- ✅ Email de confirmación al crear cita manual desde el panel.
- ✅ Envío **solo desde el servidor** (route handlers), nunca desde el cliente.
- ✅ Uso de Resend leyendo `process.env.RESEND_API_KEY` y `process.env.RESEND_FROM`
  vía POST a `https://api.resend.com/emails`.
- ✅ API key nunca expuesta al cliente ni con prefijo `NEXT_PUBLIC`.
- ✅ El email incluye nombre del negocio, servicio, fecha y hora.
- ✅ Hora de Chile (`America/Santiago`) en formato 24h, consistente con `lib/format.ts`.
- ✅ Degradación con gracia: sin API key o ante fallo de Resend, la reserva se crea
  igual y solo se registra en log.
- ✅ Se respeta el flujo RPC existente y el control de solapamiento (respuesta 409).
- ✅ No se agregaron recordatorios programados ni cron.

## PENDIENTES / Limitaciones reales

- El email es texto/HTML en español embebido en `lib/email.ts`; no hay plantillas
  externas ni sistema de internacionalización.
- `formatDate` usa locale `es-ES` (no `es-CL`); el resultado es texto de fecha en
  español, pero el locale no es estrictamente chileno.
- No hay verificación de entrega ni reintentos: ante error de Resend solo se hace
  `console.warn`. En el panel se expone `email_sent`, pero en la ruta pública no se
  retorna ese flag al cliente.
- No se enviaron pruebas automatizadas del envío de email en este run.
- El envío depende de que `customer_email` sea provisto y pase la validación de
  formato básica (`/^.+@.+\..+$/`).

## Despliegue

✅ Desplegado y verificado en Railway (build OK).
