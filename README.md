# Ecosistema de Automatización IA — Leads VIP con Propuestas (HITL)

**Nicolas Navarro** · Trabajo Final

Sistema que califica leads comerciales entrantes con IA, redacta una propuesta personalizada, y **nunca contacta a un cliente real sin que un humano la apruebe primero**.

## Stack

| Categoría | Herramienta |
|---|---|
| Orquestador | **n8n** |
| Base de datos | **Airtable** (4 tablas vinculadas: Leads, Propuestas, Errores y Dashboard) |
| Procesamiento IA | **Claude Haiku 4.5** (Anthropic API), prompt estructurado con Structured Output Parser |
| Canal de salida | **Gmail** (envío final, con Thread ID) + **Slack** (aviso HITL y respuestas en el mismo hilo) |

## Enlaces

- **Workflow en vivo (n8n):** https://nnavarro2890.app.n8n.cloud/workflow/qmwyzxEDlq4O8Y6H
- **Dashboard de control (enlace público, KPIs y tasa de errores en vivo):** https://airtable.com/app9d9WVwEBaTXKlJ/shrQnKIh87AaPgd40
- **Base de datos completa (Airtable, lectura pública — las 4 tablas):** https://airtable.com/app9d9WVwEBaTXKlJ/shrtmhdtEg23LDX4Z

## Dónde está cada criterio de la rúbrica

| Criterio | Entregable |
|---|---|
| Mapa de arquitectura | [`01_arquitectura.pdf`](01_arquitectura.pdf) |
| Estructuras de datos documentadas | [`02_manual_datos.pdf`](02_manual_datos.pdf) |
| Optimización de costos | [`03_matriz_costos.pdf`](03_matriz_costos.pdf) |
| Seguridad y resiliencia | [`04_seguridad_resiliencia.pdf`](04_seguridad_resiliencia.pdf) |
| Dashboard de control | [Enlace público al Panel de KPIs](https://airtable.com/app9d9WVwEBaTXKlJ/shrQnKIh87AaPgd40) (ver sección "Dashboard de control") |
| Archivos técnicos de respaldo | [`blueprint_raw.json`](blueprint_raw.json), link a la base en modo lectura y [`evidencia/`](evidencia/) |

## Archivos de este repo

| Archivo | Contenido |
|---|---|
| `01_arquitectura.pdf` | Diagrama lógico del flujo (2 triggers, guard anti-reprocesamiento, validación, IA, HITL, canales de salida, manejo de errores y dashboard) |
| `02_manual_datos.pdf` | Esquema de las 4 tablas de Airtable, ciclo de vida del Estado, esquemas JSON de cada integración y cobertura de los 31 nodos |
| `03_matriz_costos.pdf` | Comparativa de modelos de IA y justificación de costos, con los tokens reales medidos en las pruebas y precios vigentes de Anthropic |
| `04_seguridad_resiliencia.pdf` | Minimización de datos, credenciales, salvaguardas (guard, validación, fallos de IA, anti-loop), riesgos conocidos y puntos HITL |
| `blueprint_raw.json` | Export técnico completo del workflow de n8n (33 nodos: 31 funcionales + 2 notas), importable |
| `blueprint.json` | Versión resumida y comentada del mismo flujo, para lectura rápida |
| `evidencia/` | Capturas reales de las ejecuciones de prueba (ver tabla de abajo) |

## Arquitectura en una línea

```
Airtable (Estado=Pendiente)  ─┐
Webhook de prueba (lead_id)  ─┴→ Releer el lead en Airtable (estado actual, no el del trigger)
  → Guard anti-reprocesamiento: si ya no está "Pendiente", se omite (no se vuelve a llamar a la IA)
  → Validar datos completos (si faltan: log de error indicando qué campos, corta acá)
  → Claude Haiku 4.5 clasifica VIP + redacta propuesta en texto plano (JSON estructurado)
  → Guarda en Airtable + avisa al equipo por Slack (y guarda el hilo del aviso)
  → PAUSA (espera aprobación humana, revisa cada 1 min, máx. 5 intentos)
  → Si se aprueba: Estado "Aprobado por Humano" → envía por Gmail, guarda el Thread ID, marca "Enviado" y lo confirma en el hilo de Slack
  → Si se agotan los intentos: marca "Rechazado", registra el timeout y lo avisa en el hilo de Slack
```

## Pruebas realizadas (5, incluyendo caminos infelices)

Todas las pruebas se corrieron el 26/09/2026 sobre la versión final del workflow, desde una base limpia (tablas Leads, Propuestas y Errores vacías), con un lead por escenario (la prueba 5 reutiliza el lead de Diego), disparando cada uno por su ID a través del webhook de prueba. Los números de ejecución son globales de la instancia de n8n (compartidos con otros workflows y con corridas anteriores), por eso no empiezan en 1. De cada caso se captura el lienzo de n8n y, cuando corresponde, el hilo real de Slack y el email real en Gmail. La aprobación humana de las pruebas 2 y 3 se hizo tildando "Aprobado" en la tabla Propuestas durante la pausa.

| # | Caso | Resultado | Evidencia |
|---|---|---|---|
| 1 | Camino infeliz: datos incompletos (lead sin Email ni Mensaje Original) | La validación lo detectó *antes* de llamar a la IA (`Succeeded in 1.696s`, sin gastar en la API), registró *"Datos incompletos: falta Email, Mensaje Original"* y marcó **Estado=Error** | [Lienzo](evidencia/01_datos_incompletos_n8n_751.jpg) · [Errores](evidencia/07_airtable_errores.jpg) |
| 2 | Lead VIP (Sofía Paz, USD 6.000 + plazo ajustado) | Clasificado como **VIP** (confianza 92%), aprobado en Airtable, pasó por **Aprobado por Humano**, se envió por Gmail en texto plano y el sistema lo confirmó en el hilo de Slack — `Succeeded in 1m 10.215s` | [Lienzo 1](evidencia/02_sofia_vip_aprobado_n8n_752_parte1.jpg) · [Lienzo 2](evidencia/02_sofia_vip_aprobado_n8n_752_parte2.jpg) · [Aviso Slack](evidencia/08_slack_aviso_sofia.jpg) · [Respuesta en el hilo](evidencia/08_slack_hilo_sofia_respuesta.jpg) · [Gmail](evidencia/09_gmail_mail_sofia.jpg) |
| 3 | Lead no VIP (Carla Díaz, USD 750, sin urgencia) | Clasificado como **no VIP** (confianza 75%), aprobado y enviado por Gmail, con confirmación en el hilo de Slack — `Succeeded in 1m 10.122s` | [Lienzo 1](evidencia/03_carla_no_vip_aprobado_n8n_753_parte1.jpg) · [Lienzo 2](evidencia/03_carla_no_vip_aprobado_n8n_753_parte2.jpg) · [Hilo Slack](evidencia/08_slack_hilo_carla.jpg) · [Bandeja](evidencia/09_gmail_bandeja_propuestas.jpg) |
| 4 | Límite anti-loop-infinito (Diego Torres, VIP, sin aprobar a propósito) | El sistema cortó exactamente a los 5 intentos, marcó **Rechazado**, registró *"Se agoto el tiempo de espera de aprobacion humana (HITL) sin respuesta"* (texto literal del registro) y avisó en el hilo de Slack que no se envió nada — `Succeeded in 5m 10.233s` | [Lienzo 1](evidencia/04_diego_vip_timeout_n8n_754_parte1.jpg) · [Lienzo 2](evidencia/04_diego_vip_timeout_n8n_754_parte2.jpg) · [Hilo Slack](evidencia/08_slack_hilo_diego_timeout.jpg) |
| 5 | Guard anti-reprocesamiento (Diego Torres, ya *Rechazado*, disparado de nuevo) | El guard detectó que el lead ya no estaba en *Pendiente* y el flujo terminó en "Omitir: Lead Ya Procesado" — `Succeeded in 628ms`, sin llamar a la IA, sin crear una segunda propuesta y sin modificar el lead | [Lienzo](evidencia/12_guard_reprocesamiento_n8n_755.jpg) |

Estado final en Airtable: 4 leads, 3 propuestas (una por lead procesado, sin duplicados) y 2 errores — [Leads](evidencia/05_airtable_leads_estado_final.jpg) · [Propuestas](evidencia/06_airtable_propuestas.jpg) · [Errores](evidencia/07_airtable_errores.jpg) · [Base pública](evidencia/11_airtable_base_compartida.jpg). La bandeja de Gmail filtrada por la hora de la corrida muestra exactamente 2 propuestas enviadas (Sofía y Carla); Diego no recibió nada.

### Nota metodológica

Al ejecutar el workflow manualmente desde el editor, el trigger de Airtable **ignora la fórmula de filtro y devuelve siempre el primer registro de la tabla**, sin importar su estado. En pruebas anteriores eso hizo que un lead ya enviado se volviera a procesar (propuesta duplicada y estado sobrescrito). Se resolvió de dos maneras:

1. **Guard anti-reprocesamiento** en el propio flujo: después de cualquier trigger se relee el lead en Airtable y solo continúa si su estado actual es *Pendiente*. Protege también a producción.
2. **Webhook de prueba** (`POST /webhook/leads-vip-prueba` con `{ "lead_id": "rec..." }`): permite disparar el flujo sobre un lead puntual, sin depender de qué registro devuelva el trigger. Pasa por el mismo guard.

Las ejecuciones automáticas (poll cada 5 min) sí respetan el filtro, pero consumen el cupo de 50 ejecuciones de producción por mes del plan de n8n Cloud usado en el proyecto. Ese cupo se agotó durante el desarrollo (el banner "50/50 Executions" de las capturas), por eso las 5 pruebas documentadas se corrieron en modo manual con el webhook de prueba; hasta que el cupo se renueve, el trigger automático no procesa leads nuevos. El comportamiento queda respaldado por las capturas y por `blueprint_raw.json`, que se puede importar y ejecutar en cualquier instancia de n8n.

## Dashboard de control

**Enlace público:** https://airtable.com/app9d9WVwEBaTXKlJ/shrQnKIh87AaPgd40 (vista compartida de Airtable, solo lectura, no requiere cuenta).

Es la vista "Panel de KPIs" de la tabla **Dashboard**, cuyos indicadores se calculan en vivo con campos rollup y fórmulas sobre los datos reales:

| KPI | Cálculo |
|---|---|
| Total Leads · Leads VIP · % VIP | Cantidad de leads, cuántos marcó la IA como VIP y su porcentaje |
| Enviados · Tasa de Envío (%) | Leads con propuesta aprobada y enviada, y su porcentaje |
| Rechazados | Leads sin aprobación humana dentro del plazo |
| Leads en Error · **Tasa de Error (%)** | Leads que terminaron en Error y su porcentaje sobre el total |
| Errores Registrados | Registros en la tabla Errores (datos incompletos, fallos de IA y timeouts) |
| En Curso | Leads todavía en el pipeline (Pendiente, Procesado por IA o Aprobado por Humano) |

Cada lead nuevo se vincula solo al panel mediante la automatización de Airtable "Vincular lead nuevo al Dashboard" ([captura](evidencia/13_airtable_automatizacion_dashboard.jpg)), así los números se actualizan sin intervención manual. Valores al cierre de las pruebas: 4 leads, 2 VIP, 2 enviados, 1 rechazado, 1 en error, **tasa de error 25%** ([captura](evidencia/10_dashboard_panel_kpis_publico.jpg)). La tasa de error cuenta leads que terminaron en Estado=Error (1 de 4); "Errores Registrados" vale 2 porque el timeout de Diego también queda en la tabla Errores, aunque ese lead termina en Rechazado. Después de las pruebas se sumó un quinto lead, Lucía Gómez (VIP, aprobada y enviada), usado en el video demo; por eso el panel en vivo muestra hoy 5 leads, 3 VIP, 3 enviados y una tasa de error del 20%.

Como complemento, la base tiene también un panel en Airtable Interfaces con gráfico de distribución por Estado ([captura](evidencia/10_dashboard_interface_graficos.jpg)); ese panel no se comparte con enlace público porque Airtable solo permite publicar Interfaces desde el plan Team (USD 20 por usuario/mes con facturación anual), mientras que las vistas compartidas como el Panel de KPIs son gratuitas.

## Check de seguridad

| Pregunta | Respuesta |
|---|---|
| ¿El flujo tiene un filtro para evitar bucles infinitos? | Sí. "Filtro: ¿Excedió Intentos?" usa `$runIndex >= 4` y corta el ciclo de espera a los 5 intentos (prueba 4). Además, el guard impide reprocesar un lead que ya salió de *Pendiente* (prueba 5). |
| ¿Se comparan tipos de datos correctos en los filtros? | Sí. `faltantes.length == 0` es número contra número; `$runIndex >= 4` es número contra número con validación estricta; `Aprobado === true` es booleano; `Estado == "Pendiente"` es texto contra texto. |
| ¿El prompt de IA es dinámico y usa variables del sistema? | Sí. El mensaje al modelo se arma con expresiones que leen el registro actual de Airtable (`{{ $("Releer Lead (Estado Actual)").item.json.fields.Nombre }}`, Empresa, Presupuesto Estimado y Mensaje Original). No hay datos de clientes escritos a mano. |

## Aclaraciones de diseño

- **Modelo de IA:** se usó Claude Haiku 4.5 (real, vía API propia de Anthropic) — no un sustituto gratuito — dado que la matriz de costos necesitaba números reales. Máximo de 1024 tokens de salida.
- **Formato de la propuesta:** el prompt exige texto plano sin markdown, porque el email se envía como texto; en una versión anterior las negritas en markdown llegaban como `**asteriscos**` al cliente.
- **Hilos (Thread ID):** el `threadId` de Gmail se guarda en el campo `Leads.Message-ID` (que contiene ese threadId) para que cualquier seguimiento futuro responda al cliente en la misma conversación, y el timestamp del aviso de Slack se guarda en `Propuestas.Slack Thread TS` para que el resultado (envío o timeout) quede como respuesta en ese mismo hilo.
- **Contador de reintentos:** el límite anti-loop-infinito usa `$runIndex` (índice de corrida propio del nodo "Filtro: ¿Excedió Intentos?"), no un conteo vía `$('Pausa: Esperar Aprobación').all().length` — ese método devuelve solo los ítems de la última corrida del nodo, no un histórico, y fue corregido tras detectarse en pruebas en vivo que el loop no cortaba al límite esperado.
- **Datos frescos:** todos los nodos toman nombre, email e ID del lead desde "Releer Lead (Estado Actual)", no desde el payload del trigger, para trabajar siempre con el estado real del registro.

## Video demo

Video de 3 minutos (trigger → procesamiento en n8n → resultado final, sin mostrar credenciales): https://drive.google.com/file/d/1-JPpBT8BmHkICwzKNWpMnZatk9VBre8X/view
