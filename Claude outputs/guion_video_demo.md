# Guion del video demo (3 minutos)

Antes de grabar:

- Crear en Airtable un lead nuevo en **Pendiente** (por ejemplo, "Lucía Gómez", con Email con alias +lucia, un presupuesto de USD 5.000 y un mensaje con urgencia).
- Tener abiertas 4 pestañas: el workflow en n8n, Airtable (Leads y Propuestas), Slack (#general-nicolasnavarro) y Gmail (con el filtro por asunto "Propuesta comercial personalizada").
- Cerrar cualquier panel que muestre credenciales. No abrir la sección Credentials de n8n ni los nodos por dentro con la pestaña de credencial visible.
- Si el cupo de n8n sigue en 50/50, disparar con el webhook de prueba (ejecución manual) en lugar de esperar el trigger automático.

| Tiempo | Qué mostrar | Qué decir |
|---|---|---|
| 0:00–0:20 | README del repo en GitHub | "Sistema de leads VIP: n8n orquesta, Airtable es la memoria, Claude califica y redacta, y Gmail y Slack son la salida. Nada se envía sin aprobación humana." |
| 0:20–0:45 | Lienzo del workflow en n8n (vista general) | Recorrer de izquierda a derecha: los 2 triggers, el guard, la validación, el nodo de IA con su rama de error, la pausa HITL y las dos salidas (Gmail o timeout). |
| 0:45–1:05 | Airtable, tabla Leads | Mostrar el lead nuevo en Pendiente. "Este es el disparador: un lead en estado Pendiente." |
| 1:05–1:35 | n8n: ejecutar con el webhook de prueba y ver los nodos ponerse en verde | "Relee el lead, pasa el guard, valida los datos y Claude devuelve un JSON con VIP, confianza y propuesta, con máximo 1024 tokens." |
| 1:35–2:00 | Slack: el aviso en el canal | "El equipo recibe la propuesta completa. El flujo queda en pausa esperando." Luego, en Airtable Propuestas, tildar **Aprobado**. |
| 2:00–2:30 | Airtable Leads (Estado pasa a Aprobado por Humano y luego Enviado) y Gmail (el email recibido) | "La aprobación queda registrada, se envía el email en texto plano y se guarda el Thread ID." |
| 2:30–2:45 | Slack: la respuesta en el hilo | "El sistema confirma el envío en el mismo hilo del aviso." |
| 2:45–3:00 | Dashboard público (Panel de KPIs) | "Los KPIs y la tasa de error se actualizan solos. Si nadie aprueba en 5 minutos, el lead queda Rechazado; si faltan datos o falla la IA, queda registrado en Errores." |

Después de grabar:

- Subir el video (YouTube como "no listado" o Google Drive con acceso por enlace) y reemplazar **[agregar enlace al video]** al final del README por el link.
- Opcional: borrar el lead de prueba del video si no se quiere que cambie los números del dashboard documentados en el README (4 leads, tasa de error 25%).
