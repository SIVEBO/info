# Integraciones del Sistema SIVEBO

Este documento detalla las directrices arquitectónicas y los estándares para la integración de sistemas externos en la plataforma SIVEBO.

## 1. Estrategia de Integración

Para mantener el desacoplamiento de los microservicios y garantizar la mantenibilidad, SIVEBO utiliza un **patrón de integración centralizada**.

- **MS-11 Integrator:** Es el microservicio dedicado exclusivamente a interactuar con sistemas externos (ej. ClickUp, Trello).
- **Event Bus:** Los microservicios de dominio publican eventos (ej. `AdmissionCreatedEvent`) en un bus interno (Kafka/RabbitMQ). El `MS-11` consume estos eventos para ejecutar las acciones necesarias en el sistema externo.

## 2. Integración con ClickUp

### 2.1 Arquitectura
- **Outbound (Push):** `MS-11` recibe eventos internos -> Llama a la API de ClickUp mediante `WebClient`.
- **Inbound (Webhooks):** ClickUp envía eventos -> `MS-11` recibe -> Verifica firma -> Publica evento interno.

### 2.2 Seguridad (CRÍTICO)
**Nunca se deben hardcodear credenciales en el código fuente.**
- `CLICKUP_API_TOKEN` y `CLICKUP_WEBHOOK_SECRET` deben gestionarse mediante variables de entorno o un servicio de configuración segura (ej. HashiCorp Vault).

### 2.3 Resiliencia y Monitoreo
- Implementar **Resilience4j** en todas las llamadas salientes a la API de ClickUp (Retries, Circuit Breaker).
- Loguear todas las peticiones y respuestas (excluyendo datos sensibles como tokens) para propósitos de auditoría y diagnóstico.

## 3. Integración con Trello

### 3.1 Arquitectura
- **Outbound (Push):** `MS-11` consume eventos internos -> Llama a la API de Trello mediante `WebClient` configurado.
- **Inbound (Webhooks):** Trello envía eventos -> `MS-11` recibe -> Verifica firma -> Publica evento interno.

### 3.2 Credenciales
- Se requieren `TRELLO_API_KEY` y `TRELLO_API_TOKEN`. Deben gestionarse estrictamente mediante variables de entorno o un servicio de configuración segura.

## 4. Guía para el Desarrollador

1.  **Nuevo flujo de integración:**
    - Identificar el evento de dominio que dispara la acción.
    - Crear o actualizar el DTO en el `MS-11`.
    - Implementar el consumidor del evento en `MS-11`.
    - Configurar la llamada a la API en `MS-11` utilizando el `WebClient` configurado.
2.  **Configuración de Webhooks:**
    - Registrar el endpoint en la plataforma externa (ClickUp/Trello) manualmente o mediante automatización al iniciar el servicio.
    - Implementar la verificación del header de firma correspondiente (`X-Signature` o similar) en el controlador de Webhooks de `MS-11`.
