# Asistente de Expedientes — Pre-Entrega 1 (Módulo 1)

AI Automation Avanzado · CoderHouse · Comisión 97500

> **Entregas en este repositorio**
> - Pre-Entrega 1 (Módulo 1): este README y `checkpoint1_estanislao_ortiz.json`, en la raíz.
> - Pre-Entrega 4 (Módulo 4, integraciones CRM + correo + Slack): carpeta [`pre-entrega-4/`](pre-entrega-4/).

Agente de n8n que responde consultas del personal de un estudio jurídico sobre los expedientes registrados en su índice. Es la primera versión del proyecto integrador: en los próximos módulos suma multi-agente, memoria, integraciones, RAG y voz.

**Todos los datos son ficticios.** Ningún expediente, carátula ni persona es real.

## Archivo entregado

`checkpoint1_estanislao_ortiz.json` — workflow exportado de n8n Cloud.

## Arquitectura

```
                                       ┌──► Respuesta al usuario (Chat · Send a message)
When chat message received ──► AI Agent┤
                                │      └──► Log de observabilidad (Gmail)
                                │
              OpenAI Chat Model (gpt-4o) · Tool: Consultar_indice_expedientes (Google Sheets, solo lectura)
```

| Requisito | Cómo se cumple |
|---|---|
| Trigger | Chat Trigger (`When chat message received`) |
| AI Agent en modo Tools Agent | Nodo AI Agent **versión 3.1**: en esta versión Tools Agent es el único modo, por eso el nodo no muestra el selector |
| Modelo | `OpenAI Chat Model` con **gpt-4o** (temperatura 0.2) |
| Guardrail de iteraciones | **Max Iterations = 5** |
| System Prompt modular | Rol → Ámbito → Objetivo → Reglas → Escalamiento, con lo que el agente **nunca** hace: opinión o estrategia jurídica, cálculo de plazos procesales, modificar el índice, inventar datos |
| Herramienta lateral | `Consultar_indice_expedientes` (Google Sheets · Get Row(s)), conectada en la entrada *Tool* del agente, con Description extensa de cuándo usarla y cuándo no |
| Observabilidad | Nodo final **Gmail** que envía en cada ejecución: consulta, respuesta, herramientas usadas, cantidad de llamadas, si derivó a un abogado y el número de ejecución para abrir el Execution Log. Usa *Return Intermediate Steps* del AI Agent |

**Por qué hay un nodo "Respuesta al usuario":** con Gmail como último nodo, el chat mostraba la respuesta de Gmail en lugar de la del agente. El Chat Trigger está en *Using Response Nodes* y la respuesta sale de un nodo Chat en una rama paralela; Gmail sigue siendo el nodo final de su rama.

## Cómo probarlo

1. Importar el `.json` en n8n (**Workflows → Import from File**).
2. Reconectar las credenciales propias: OpenAI, Google Sheets y Gmail.
3. Crear una planilla de Google Sheets con el índice ficticio (columnas `id_expediente`, `caratula`, `fuero`, `juzgado`, `estado`, `ultima_actuacion`, `fecha_ultima_actuacion`, `proxima_actuacion`, `fecha_vencimiento`, `abogado_responsable`, `observaciones`) y elegirla en el nodo `Consultar_indice_expedientes`.
4. En `Log de observabilidad`, poner un correo propio en **To** (el exportado tiene un correo de ejemplo).
5. Abrir el chat y probar, por ejemplo:
   - `Hola` → saluda, no usa la herramienta.
   - `¿Qué vence esta semana?` → usa la herramienta y lista por fecha.
   - `¿Me conviene apelar en el exp_p3x8n1?` → deriva al abogado responsable.
   - `¿Qué es una medida cautelar?` → fuera de ámbito, no usa la herramienta.

## Validación realizada

10 pruebas con respuestas esperadas definidas antes de correrlas (saludo, expediente puntual, vencimientos de la semana, expediente inexistente, sin vencimiento, consulta ambigua, opinión jurídica, cálculo de plazo, modificación del índice y fuera de ámbito). Recorrido en verde y Execution Log auditado: el agente usa la herramienta solo en las consultas sobre expedientes, y el correo de observabilidad llega con todas las expresiones resueltas.
