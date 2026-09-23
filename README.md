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

Todas las pruebas se rehicieron desde una base limpia (un lead a la vez, sin datos de pruebas previas mezclados), con captura de cada paso: el lienzo de n8n, el mensaje real de Slack, y — cuando corresponde — el email real en Gmail.

| # | Caso | Resultado | Evidencia |
|---|---|---|---|
| 1 | Lead completo (Ana Martínez), presupuesto bajo, sin urgencia | Clasificado correctamente como **no VIP**, aprobado y enviado por Gmail con éxito | [Lienzo](evidencia/t1_01_lienzo_completo.png) · [Slack](evidencia/t1_02_slack_mensaje.png) · [Gmail](evidencia/t1_03_gmail_enviado.png) |
| 2 | Lead completo (Roberto Fernández), presupuesto alto + urgencia | Clasificado correctamente como **VIP**, aprobado y enviado por Gmail con éxito | [Pausa](evidencia/t2_01_lienzo_pausa.png) · [Slack](evidencia/t2_02_slack_mensaje.png) · [Final](evidencia/t2_03_lienzo_final.png) · [Gmail](evidencia/t2_04_gmail_enviado.png) |
| 3 | Loop HITL real (Laura Giménez) | El lead quedó sin aprobar durante **5 ciclos completos** (llegó justo al límite) antes de aprobarse — `Succeeded in 5m 11.8s` | [Pausa](evidencia/t3_01_lienzo_pausa.png) · [Slack](evidencia/t3_02_slack_mensaje.png) · [5 ciclos](evidencia/t3_03_lienzo_5_ciclos.png) · [Gmail](evidencia/t3_04_gmail_enviado.png) |
| 4 | Guarda anti-loop-infinito (Martín Ríos) | Lead dejado sin aprobar a propósito — el sistema cortó exactamente a los 5 intentos, marcó **Rechazado** y registró *"Se agotó el tiempo de espera de aprobación humana (HITL) sin respuesta"* | [Pausa](evidencia/t4_01_lienzo_pausa.png) · [Slack](evidencia/t4_02_slack_mensaje.png) · [Rechazado](evidencia/t4_03_lienzo_rechazado_timeout.png) |
| 5 | Camino infeliz: datos incompletos | Lead sin Email ni Mensaje Original — la validación lo detectó *antes* de llamar a la IA (`Succeeded in 2.059s`, sin gastar en la API), registró el error en Airtable y marcó **Estado=Error** | [Lienzo](evidencia/t5_01_lienzo_error_datos.png) |

### Nota metodológica

El botón "Execute workflow" del editor de n8n no siempre respeta el filtro configurado del trigger: en varias corridas volvió a procesar el último lead modificado en vez del nuevo. Se resolvió limpiando la tabla Leads antes de cada prueba (dejando un único registro "Pendiente" por vez), lo que garantiza que la ejecución solo puede tomar ese lead. Las ejecuciones automáticas (el poll cada 5 min) sí respetan el filtro correctamente, pero consumen la cuota de 50 ejecuciones/mes del plan gratuito — las manuales desde el editor no la consumen.

### Nota sobre los links de la base

Se dejan dos enlaces: el primero da acceso público de lectura a la base completa (las 3 tablas: Leads, Propuestas y Errores), y el segundo es la vista agrupada por Estado dentro de Leads, útil como cuadro de mando rápido.

## Aclaraciones de diseño

- **Modelo de IA:** se usó Claude Haiku 4.5 (real, vía API propia de Anthropic) — no un sustituto gratuito — dado que la matriz de costos necesitaba números reales.
- **Contador de reintentos:** la guarda anti-loop-infinito usa `$runIndex` (índice de corrida propio del nodo Filtro), no un conteo vía `$('Pausa').all().length` — ese método devuelve solo los ítems de la última corrida del nodo, no un histórico, y fue corregido tras detectarse en pruebas en vivo que el loop no cortaba al límite esperado.
- **Video demo:** no incluido en este repositorio (a grabar por separado, mostrando trigger → procesamiento → resultado, sin exponer credenciales).
