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

- **Workflow en vivo (n8n):** https://nnavarro2890.app.n8n.cloud/workflow/b3ZTGcvM6Ib7CTpf
- **Base de datos completa (Airtable, lectura pública — las 3 tablas):** https://airtable.com/app9d9WVwEBaTXKlJ/shrtmhdtEg23LDX4Z
- **Dashboard / vista agrupada por Estado (tabla Leads):** https://airtable.com/app9d9WVwEBaTXKlJ/shrq2kMKprjir606a

## Archivos de este repo

| Archivo | Contenido |
|---|---|
| `01_arquitectura.pdf` | Diagrama visual completo del flujo (triggers, IA, HITL, canales de salida, manejo de errores) |
| `02_manual_datos.pdf` | Esquema de las 3 tablas de Airtable + esquemas JSON de cada integración |
| `03_matriz_costos.pdf` | Comparativa de modelos de IA y justificación de costos, con números reales de las pruebas |
| `04_seguridad_resiliencia.pdf` | Minimización de datos, rutas de error, y explicación de los puntos HITL |
| `blueprint_raw.json` | Export técnico completo del workflow de n8n (21 nodos), importable |
| `blueprint.json` | Versión resumida y comentada del mismo flujo, para lectura rápida |
| `evidencia/` | Capturas reales de las ejecuciones de prueba (ver tabla de abajo) |

## Arquitectura en una línea

```
Airtable (Estado=Pendiente)
  → Validar datos completos (si faltan: log de error, corta acá)
  → Claude Haiku 4.5 clasifica VIP + redacta propuesta (JSON estructurado)
  → Guarda en Airtable + notifica al equipo por Slack
  → PAUSA (espera aprobación humana, revisa cada 1 min, máx. 5 intentos)
  → Si se aprueba: envía por Gmail, guarda el Thread ID, marca "Enviado"
  → Si se agotan los intentos: marca "Rechazado" y registra el timeout
```

## Pruebas realizadas (5, incluyendo camino infeliz)

| # | Caso | Resultado | Evidencia |
|---|---|---|---|
| 1 | Lead completo, presupuesto bajo, sin urgencia | Clasificado correctamente como **no VIP**, propuesta generada, aprobado y enviado por Gmail con éxito | [`evidencia/01_test1_ana_martinez_no_vip.png`](evidencia/01_test1_ana_martinez_no_vip.png) |
| 2 | Lead completo, presupuesto alto + urgencia | Clasificado correctamente como **VIP**, aprobado y enviado por Gmail con éxito | [`evidencia/02_test2_roberto_vip_loop_x2.png`](evidencia/02_test2_roberto_vip_loop_x2.png) |
| 3 | Loop HITL real | El lead quedó sin aprobar durante 2 ciclos completos (confirmado "✓2" en n8n) antes de aprobarse | [`evidencia/03_loop_hitl_vuelve_a_pausa.png`](evidencia/03_loop_hitl_vuelve_a_pausa.png) |
| 4 | Guarda anti-loop-infinito | Lead dejado sin aprobar a propósito — el sistema cortó exactamente a los 5 intentos (`$runIndex >= 4`), marcó **Rechazado** y registró el timeout | [`evidencia/05_test4_timeout_5_ciclos.png`](evidencia/05_test4_timeout_5_ciclos.png) (ejecución real, `Succeeded in 5m 11s`) |
| 5 | Camino infeliz: datos incompletos | Lead sin Email ni Mensaje Original — la validación lo detectó *antes* de llamar a la IA, registró el error en Airtable (vinculado al lead) y marcó **Estado=Error**, sin gastar en la API | Confirmado vía API de n8n (ejecución #463, pin data) — ver nota metodológica |

### Nota metodológica

Para la prueba #5 se usó la función de *pin data* de n8n (inyección directa de datos de prueba en el nodo Trigger) en lugar de depender del disparador real, ya que el botón "Test workflow" del editor recupera el registro modificado más recientemente — no necesariamente el que se quiere probar. Esto permitió validar el camino infeliz de forma determinística, con escrituras reales en Airtable, confirmadas por API. No se pudo obtener además una captura de pantalla fresca de esta prueba puntual: al reintentarlo, la cuota de ejecuciones automáticas del plan gratuito de n8n (50/50) se agotó y bloqueó los intentos posteriores — un límite real de la plataforma, no del diseño del flujo.

### Nota sobre los links de la base

Se dejan dos enlaces: el primero da acceso público de lectura a la base completa (las 3 tablas: Leads, Propuestas y Errores), y el segundo es la vista agrupada por Estado dentro de Leads, útil como cuadro de mando rápido.

## Aclaraciones de diseño

- **Modelo de IA:** se usó Claude Haiku 4.5 (real, vía API propia de Anthropic) — no un sustituto gratuito — dado que la matriz de costos necesitaba números reales.
- **Contador de reintentos:** la guarda anti-loop-infinito usa `$runIndex` (índice de corrida propio del nodo Filtro), no un conteo vía `$('Pausa').all().length` — ese método devuelve solo los ítems de la última corrida del nodo, no un histórico, y fue corregido tras detectarse en pruebas en vivo que el loop no cortaba al límite esperado.
- **Video demo:** no incluido en este repositorio (a grabar por separado, mostrando trigger → procesamiento → resultado, sin exponer credenciales).
