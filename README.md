# Flujo A [LISTO v3] - Validación y Logs

Este es un flujo de n8n diseñado para recibir datos a través de un webhook, validarlos, y luego almacenar la información en una base de datos Supabase. También se implementa un sistema de logging para registrar tanto los éxitos como los errores durante el proceso.

## Descripción del Flujo

El flujo consta de los siguientes nodos:

*   **Webhook (a1):**
    *   **Tipo:** `n8n-nodes-base.webhook`
    *   **Método HTTP:** `POST`
    *   **Ruta:** `/contacto`
    *   **Descripción:** Actúa como punto de entrada para recibir datos externos. Espera una solicitud POST en la ruta `/contacto`.

*   **Validar (a2):**
    *   **Tipo:** `n8n-nodes-base.code`
    *   **Descripción:** Este nodo procesa los datos recibidos del webhook y realiza las siguientes validaciones:
        *   Verifica que `event_id` esté presente y no vacío.
        *   Valida que `nombre` tenga entre 2 y 80 caracteres.
        *   Comprueba que `email` tenga un formato válido.
        *   Asegura que `mensaje` tenga un mínimo de 10 caracteres.
        *   Trunca el mensaje a un máximo de 2000 caracteres si excede este límite, y establece un flag `truncado`.
        *   Determina si los datos son `valido` (basado en si hay errores).
        *   Agrega los errores encontrados en una lista.
        *   Registra la fecha `recibido_en`.
        *   Asigna un valor por defecto `'web'` a `origen` si no se proporciona.

*   **Valido? (a3):**
    *   **Tipo:** `n8n-nodes-base.if`
    *   **Condición:** Comprueba si el valor `{{ $json.valido }}` es `true`.
    *   **Descripción:** Dirige el flujo a la rama de éxito si los datos son válidos, o a la rama de error si no lo son.

*   **Guardar contacto (a4):**
    *   **Tipo:** `n8n-nodes-base.httpRequest`
    *   **Método HTTP:** `POST`
    *   **URL:** `https://bohfyoaabfvmqfizvkin.supabase.co/rest/v1/contactos`
    *   **Headers:** Incluye `apikey`, `Authorization`, `Content-Type` y `Prefer` para interactuar con la API de Supabase.
    *   **Cuerpo (JSON):** Envía los datos validados (`event_id`, `nombre`, `email`, `mensaje`, `origen`, `truncado`) a la tabla `contactos` en Supabase.
    *   **Retry:** Configurado para reintentar la solicitud hasta 3 veces en caso de fallo, con esperas de 2 segundos entre intentos.
    *   **Descripción:** Almacena los datos de contacto válidos en la base de datos Supabase.

*   **Responder 200 (a5):**
    *   **Tipo:** `n8n-nodes-base.respondToWebhook`
    *   **Respuesta:** Devuelve un JSON `{ ok: true, event_id: ... }` con un código de estado HTTP `200`.
    *   **Descripción:** Confirma al emisor del webhook que la solicitud fue procesada exitosamente.

*   **Armar error 400 (a6):**
    *   **Tipo:** `n8n-nodes-base.code`
    *   **Descripción:** Prepara la estructura de la respuesta de error. Formatea los mensajes de error para ser enviados de vuelta al cliente.

*   **Responder 400 (a7):**
    *   **Tipo:** `n8n-nodes-base.respondToWebhook`
    *   **Respuesta:** Devuelve un JSON `{ ok: false, errores: ..., event_id: ..., recibido_en: ... }` con un código de estado HTTP `400`.
    *   **Descripción:** Informa al emisor del webhook que hubo errores de validación en los datos proporcionados.

*   **Log error validacion (a8):**
    *   **Tipo:** `n8n-nodes-base.httpRequest`
    *   **Método HTTP:** `POST`
    *   **URL:** `https://bohfyoaabfvmqfizvkin.supabase.co/rest/v1/logs_ejecucion`
    *   **Headers:** Incluye `apikey`, `Authorization`, `Content-Type` para interactuar con Supabase.
    *   **Cuerpo (JSON):** Registra el `event_id`, el resultado como `'error_validacion'`, el `paso_fallido` (`'Validar'`) y los `mensaje_error` en la tabla `logs_ejecucion` de Supabase.
    *   **Descripción:** Guarda un registro detallado del error de validación en la base de datos.

## Conexiones

1.  El Webhook se conecta a Validar.
2.  Validar se conecta a Valido?.
3.  Valido? se bifurca:
    *   La rama `true` se conecta a Guardar contacto.
    *   La rama `false` se conecta a Armar error 400.
4.  Guardar contacto se conecta a Responder 200.
5.  Armar error 400 se conecta a Log error validacion.
6.  Log error validacion se conecta a Responder 400.
7.  Responder 400 no tiene salidas posteriores en este flujo.

## Detalles de Configuración

*   **Nombre del Flujo:** `Flujo A [LISTO v3] - validacion y logs`
*   **Versión de Ejecución:** `v1`
*   **Claves de API y URLs de Supabase:** Las credenciales de Supabase están hardcodeadas en los nodos `httpRequest`. Se recomienda gestionar estas credenciales de forma más segura (ej. mediante variables de entorno o un nodo de autenticación dedicado si n8n lo permite de forma segura).
*   **Lógica de Validación:** Detallada en la descripción del nodo "Validar".

## Consideraciones de Seguridad

Las claves de API y las URL de Supabase están expuestas directamente en la configuración del flujo. Para entornos de producción, se recomienda encarecidamente utilizar métodos más seguros para manejar credenciales sensibles, como variables de entorno o secretos gestionados por la plataforma donde se ejecuta n8n.
