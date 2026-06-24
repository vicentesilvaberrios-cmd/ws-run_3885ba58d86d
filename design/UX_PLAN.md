# Plan de UX — Confirmación por email al reservar (fix)

Alcance: **NO** se agregan pantallas nuevas. El fix es backend (envío de email vía Resend al crear reserva). Este plan cubre **únicamente** los puntos de UX visibles para el usuario en los dos flujos de reserva existentes.

---

## 1. Reserva pública del cliente

- **Ruta:** `app/book/[slug]/page.tsx`
- **Objetivo:** el cliente elige servicio, fecha y hora y completa la reserva desde el link público del negocio.
- **Layout:** contenedor centrado (`.container`), tarjeta (`.card`) con formulario en stack (`.stack`). Móvil primero.

### Componentes / clases a usar
- `.card` envolviendo todo el formulario.
- `.field` + `.input` para nombre, email, teléfono, servicio, fecha, hora.
- `.btn .btn-primary` para "Reservar".
- `.alert .alert-success` y `.alert .alert-error` para feedback.
- `.empty-state` si el negocio no tiene servicios configurados.

### Copy de cara al usuario (es-ES, sin jerga)
- **Título (h1):** "Reservar en {nombre del negocio}"
- **Etiqueta botón principal:** "Confirmar reserva"
- **Estado cargando del envío:** "Confirmando tu reserva…"
- **Mensaje de éxito (tras reservar):**
  - Título: "¡Reserva confirmada!"
  - Cuerpo: "Te enviamos un correo de confirmación a **{email}** con los detalles de tu cita. Si no lo ves, revisa la bandeja de no deseados."
  - Botón secundario: "Hacer otra reserva"
- **Mensaje de éxito (si el email NO pudo enviarse):**
  - Título: "¡Reserva confirmada!"
  - Cuerpo: "Tu cita quedó registrada. Intentaremos enviarte la confirmación por correo en unos minutos."
  - (Variante del `.alert-success`, NO mostrar error — la reserva sí se creó.)
- **Mensaje de error al reservar:** "No pudimos completar tu reserva. Inténtalo de nuevo."
  - Botón: "Reintentar"
- **Validación email:** "Escribe un correo válido para enviarte la confirmación."

### Responsive
- Inputs a ancho completo en móvil; grid `.grid .grid-sm-2` para fecha/hora a partir de 640px.

### Accesibilidad
- Cada input con su `<label htmlFor>`.
- Email inválido → `aria-invalid="true"` + `.error-text` con `aria-describedby`.
- El mensaje de éxito debe ser `role="status"` para lectores de pantalla.
- Foco se mueve al heading de éxito tras reservar.

---

## 2. Creación manual de cita desde el panel (Agenda)

- **Ruta afectada:** `app/dashboard/agenda/page.tsx` (modal/formulario de "Nueva cita").
- **Objetivo:** el dueño/operador crea una cita a nombre de un cliente desde el panel.
- **Layout:** modal o panel lateral (`.card` + `.stack`). Si ya existe el formulario, **solo ajustar feedback**, no rediseñar.

### Componentes / clases a usar
- `.card` + `.field` + `.input` para cliente, servicio, fecha, hora.
- `.btn .btn-primary` → "Guardar cita".
- `.alert .alert-success` para confirmación.
- `.alert .alert-error` para fallo.

### Copy
- **Título modal:** "Nueva cita" / "Agendar cita para {cliente}".
- **Botón:** "Guardar cita".
- **Éxito:** "Cita guardada. Se envió la confirmación al correo del cliente."
- **Éxito sin email:** "Cita guardada. No pudimos enviar el correo de confirmación; avisa al cliente directamente."
- **Error:** "No pudimos guardar la cita. Revisa los datos e inténtalo de nuevo."
- **Estado cargando:** "Guardando cita…"

### Responsive
- Modal/pasa a bottom-sheet en móvil, manteniendo `.card`.

### Accesibilidad
- Trampa de foco dentro del modal mientras esté abierto.
- Foco vuelve al botón "Nueva cita" al cerrar.
- Errores de campo con `aria-invalid` + `.error-text`.

---

## 3. Contenido del email (texto del correo)

No es UI de la app, pero el copy debe ser consistente y en español:

- **Asunto:** "Tu reserva en {nombre del negocio} — {fecha} {hora}"
- **Cuerpo (texto plano):**
  - "Hola {nombre del cliente},"
  - "Tu cita quedó confirmada:"
  - "• Negocio: {nombre del negocio}"
  - "• Servicio: {servicio}"
  - "• Fecha: {fecha en formato DD/MM/YYYY}"
  - "• Hora: {HH:mm} (hora de Chile)"
  - "Si necesitas cambiar la hora, responde este correo."
  - Firma: "— {nombre del negocio}"

> La fecha/hora debe ir en **hora de Chile, 24h**, mismo formato que ya usa la app.

---

## 4. Notas de implementación (para devs, no UI)

- Envío **solo server-side** (route handler o server action). Nunca exponer `RESEND_API_KEY`.
- **Degradar con gracia:** si `RESEND_API_KEY` no existe o Resend falla, la reserva se crea igual; solo registrar log y mostrar variante "sin email" del mensaje de éxito.
- Mantener `no_overlap_booked`, RLS y `org_id` como están.
- No agregar recordatorios programados (fuera de alcance).
