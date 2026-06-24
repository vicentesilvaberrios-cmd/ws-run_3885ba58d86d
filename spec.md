# Solicitud de cambios (modo fix)

Momo (FIX incremental). NO reescribas lo existente; respeta el esquema, RLS/multi-tenant (org_id), el estilo, el constraint no_overlap_booked y las fichas de cliente. Agrega SOLO el EMAIL DE CONFIRMACION al reservar:

1. Al crear una reserva (desde la pagina publica del cliente y al crear cita manual desde el panel), enviar un email de confirmacion al email del cliente. SOLO desde el servidor (route handler o server action), nunca desde el cliente.
2. Usar Resend leyendo process.env.RESEND_API_KEY y process.env.RESEND_FROM (POST al endpoint de emails de Resend o el SDK resend). NUNCA expongas la API key al cliente ni uses el prefijo NEXT_PUBLIC para ella.
3. El email incluye: nombre del negocio, servicio, fecha y hora de la cita en hora de Chile y formato 24h (consistente con el resto de la app).
4. DEGRADAR CON GRACIA: si RESEND_API_KEY no esta disponible, la reserva DEBE crearse igual (no rompas el flujo); solo registra que el email no se envio.

NO agregues recordatorios programados (requieren un cron y van aparte). Manten el alcance acotado al email de confirmacion en el momento de reservar. Respeta la regla de dependencias seguras.