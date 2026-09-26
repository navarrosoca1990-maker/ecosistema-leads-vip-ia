# Ecosistema de Automatización IA — Leads VIP con Propuestas (HITL)

**Nicolas Navarro** · Trabajo Final

Sistema que califica leads comerciales entrantes con IA, redacta una propuesta personalizada, y **nunca contacta a un cliente real sin que un humano la apruebe primero**.

## Stack

| Categoría | Herramienta |
|---|---|
| Orquestador | **n8n** |
| Base de datos | **Airtable** (3 tablas vinculadas: Leads, Propuestas, Errores) |
| Procesamiento IA | **Claude Haiku 4.5** (Anthropic API), prompt estructurado con Structured Output Parser |
| Canal de salida | **Gmail** (envío final, con Thread ID) + **Slack** (notificación HITL) |

## Enlaces

- **Workflow en vivo (n8n):** https://nnavarro2890.app.n8n.cloud/workflow/qmwyzxEDlq4O8Y6H
- **Base de datos completa (Airtable, lectura pública — las 3 tablas):** https://airtable.com/app9d9WVwEBaTXKlJ/shrtmhdtEg23LDX4Z

## Archivos de este repo

| Archivo | Contenido |
|---|---|
| `01_arquitectura.pdf` | Diagrama lógico del flujo (2 triggers, guard anti-reprocesamiento, validación, IA, HITL, canales de salida, manejo de errores) |
| `02_manual_datos.pdf` | Esquema de las 3 tablas de Airtable, ciclo de vida del Estado, esquemas JSON de cada integración y cobertura de los 27 nodos |
| `03_matriz_costos.pdf` | Comparativa de modelos de IA y justificación de costos, con los tokens reales medidos en las pruebas y precios vigentes de Anthropic |
| `04_seguridad_resiliencia.pdf` | Minimización de datos, credenciales, salvaguardas (guard, validación, fallos de IA, anti-loop), riesgos conocidos y puntos HITL |
| `blueprint_raw.json` | Export técnico completo del workflow de n8n (29 nodos: 27 funcionales + 2 notas), importable |
| `blueprint.json` | Versión resumida y comentada del mismo flujo, para lectura rápida |
| `evidencia/` | Capturas reales de las ejecuciones de prueba (ver tabla de abajo) |

## Arquitectura en una línea

```
Airtable (Estado=Pendiente)  ─┐
Webhook de prueba (lead_id)  ─┴→ Releer el lead en Airtable (estado actual, no el del trigger)
  → Guard anti-reprocesamiento: si ya no está "Pendiente", se omite (no se vuelve a llamar a la IA)
  → Validar datos completos (si faltan: log de error indicando qué campos, corta acá)
  → Claude Haiku 4.5 clasifica VIP + redacta propuesta en texto plano (JSON estructurado)
  → Guarda en Airtable + notifica al equipo por Slack
  → PAUSA (espera aprobación humana, revisa cada 1 min, máx. 5 intentos)
  → Si se aprueba: envía por Gmail, guarda el Thread ID, marca "Enviado"
  → Si se agotan los intentos: marca "Rechazado" y registra el timeout
```

## Pruebas realizadas (5, incluyendo caminos infelices)

Todas las pruebas se corrieron el 26/09/2026 sobre el workflow v2, desde una base limpia (tablas Leads, Propuestas y Errores vacías), con un lead por escenario (la prueba 5 reutiliza el lead de Diego), disparando cada uno por su ID a través del webhook de prueba. Los números de ejecución son globales de la instancia de n8n (compartidos con otros workflows), por eso no son correlativos. De cada caso se captura el lienzo de n8n y, cuando corresponde, el mensaje real de Slack y el email real en Gmail.

| # | Caso | Resultado | Evidencia |
|---|---|---|---|
| 1 | Camino infeliz: datos incompletos (lead sin Email ni Mensaje Original) | La validación lo detectó *antes* de llamar a la IA (`Succeeded in 1.733s`, sin gastar en la API), registró *"Datos incompletos: falta Email, Mensaje Original"* y marcó **Estado=Error** | [Lienzo](evidencia/01_datos_incompletos_n8n_744.jpg) · [Errores](evidencia/07_airtable_errores.jpg) |
| 2 | Lead VIP (Sofía Paz, USD 6.000 + plazo ajustado) | Clasificado como **VIP** (confianza 92%), aprobado en Airtable y enviado por Gmail en texto plano — `Succeeded in 2m 9s` | [Lienzo 1](evidencia/02_sofia_vip_aprobado_n8n_745_parte1.jpg) · [Lienzo 2](evidencia/02_sofia_vip_aprobado_n8n_745_parte2.jpg) · [Slack](evidencia/08_slack_aviso_sofia.jpg) · [Gmail](evidencia/09_gmail_mail_sofia.jpg) |
| 3 | Lead no VIP (Carla Díaz, USD 750, sin urgencia) | Clasificado como **no VIP** (confianza 75%), aprobado y enviado por Gmail — `Succeeded in 2m 9s` | [Lienzo 1](evidencia/03_carla_no_vip_aprobado_n8n_746_parte1.jpg) · [Lienzo 2](evidencia/03_carla_no_vip_aprobado_n8n_746_parte2.jpg) · [Slack](evidencia/08_slack_aviso_carla.jpg) · [Bandeja](evidencia/09_gmail_bandeja_propuestas.jpg) (solo los 2 correos de las 13:00 son de esta corrida; los demás son de corridas anteriores) |
| 4 | Límite anti-loop-infinito (Diego Torres, VIP, sin aprobar a propósito) | El sistema cortó exactamente a los 5 intentos, marcó **Rechazado** y registró *"Se agotó el tiempo de espera de aprobación humana (HITL) sin respuesta"* — `Succeeded in 5m 10s` | [Lienzo 1](evidencia/04_diego_vip_timeout_n8n_747_parte1.jpg) · [Lienzo 2](evidencia/04_diego_vip_timeout_n8n_747_parte2.jpg) · [Slack](evidencia/08_slack_aviso_diego.jpg) |
| 5 | Guard anti-reprocesamiento (Diego Torres, ya *Rechazado*, disparado de nuevo) | El guard detectó que el lead ya no estaba en *Pendiente* y el flujo terminó en "Omitir: Lead Ya Procesado" — `Succeeded in 628ms`, sin llamar a la IA, sin crear una segunda propuesta y sin modificar el lead | [Lienzo](evidencia/12_guard_reprocesamiento_n8n_750.jpg) |

Estado final en Airtable: 4 leads, 3 propuestas (una por lead procesado, sin duplicados) y 2 errores — [Leads](evidencia/05_airtable_leads_estado_final.jpg) · [Propuestas](evidencia/06_airtable_propuestas.jpg) · [Errores](evidencia/07_airtable_errores.jpg) · [Base pública](evidencia/11_airtable_base_compartida.jpg).

### Nota metodológica

Al ejecutar el workflow manualmente desde el editor, el trigger de Airtable **ignora la fórmula de filtro y devuelve siempre el primer registro de la tabla**, sin importar su estado. En pruebas anteriores eso hizo que un lead ya enviado se volviera a procesar (propuesta duplicada y estado sobrescrito). Se resolvió de dos maneras:

1. **Guard anti-reprocesamiento** en el propio flujo: después de cualquier trigger se relee el lead en Airtable y solo continúa si su estado actual es *Pendiente*. Protege también a producción.
2. **Webhook de prueba** (`POST /webhook/leads-vip-prueba` con `{ "lead_id": "rec..." }`): permite disparar el flujo sobre un lead puntual, sin depender de qué registro devuelva el trigger. Pasa por el mismo guard.

Las ejecuciones automáticas (poll cada 5 min) sí respetan el filtro, pero consumen el cupo de 50 ejecuciones/mes del plan de n8n Cloud usado en el proyecto.

## Dashboard de control

Se construyó un panel real en Airtable Interfaces con números clave (Total de Leads, Leads VIP, Total de errores), un gráfico de distribución por Estado, y grillas de leads recientes y log de errores.

**No se incluye como link público** porque compartir una Interface de Airtable fuera de la organización es una función exclusiva de los planes pagos (desde Team, USD 20 por usuario/mes con facturación anual) — el plan gratuito permite compartir la base en modo lectura, pero no el dashboard de Interfaces. En su lugar, se deja como evidencia (con los datos de esta corrida):

- [`evidencia/10_dashboard_kpis_grafico.jpg`](evidencia/10_dashboard_kpis_grafico.jpg) — KPIs (Total de Leads, Leads VIP) + gráfico de distribución por Estado
- [`evidencia/10_dashboard_errores.jpg`](evidencia/10_dashboard_errores.jpg) — Total de errores registrados + log de errores

(El enlace público a la base de la sección "Enlaces" es el sustituto funcional sin costo: no tiene gráficos, pero sí los datos reales, la vista agrupada por Estado y las 3 tablas.)

## Aclaraciones de diseño

- **Modelo de IA:** se usó Claude Haiku 4.5 (real, vía API propia de Anthropic) — no un sustituto gratuito — dado que la matriz de costos necesitaba números reales.
- **Formato de la propuesta:** el prompt exige texto plano sin markdown, porque el email se envía como texto; en una versión anterior las negritas en markdown llegaban como `**asteriscos**` al cliente.
- **Contador de reintentos:** el límite anti-loop-infinito usa `$runIndex` (índice de corrida propio del nodo "Filtro: ¿Excedió Intentos?"), no un conteo vía `$('Pausa: Esperar Aprobación').all().length` — ese método devuelve solo los ítems de la última corrida del nodo, no un histórico, y fue corregido tras detectarse en pruebas en vivo que el loop no cortaba al límite esperado.
- **Datos frescos:** todos los nodos toman nombre, email e ID del lead desde "Releer Lead (Estado Actual)", no desde el payload del trigger, para trabajar siempre con el estado real del registro.
- **Video demo:** no incluido en este repositorio (a grabar por separado, mostrando trigger → procesamiento → resultado, sin exponer credenciales).
