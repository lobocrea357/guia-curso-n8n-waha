Aquí tienes el documento único en formato Markdown, listo para copiar y pegar como `README.md`. Contiene la guía completa de instalación, explicación de componentes y la configuración de Webhooks entre WAHA y n8n.

````markdown
# 🚀 Guía Completa: WAHA + n8n con Docker Compose

Esta guía explica paso a paso cómo instalar **WAHA** (WhatsApp HTTP API) y **n8n** en un mismo servidor usando Docker Compose. Incluye la explicación de cada componente, la protección con credenciales y la conexión entre ellos para automatizar mensajes de WhatsApp desde n8n.

---

## 1. 💡 Conceptos Fundamentales

### ¿Qué es WAHA?

**WAHA (WhatsApp HTTP API)** es un servicio que se conecta a tu cuenta de WhatsApp a través de WhatsApp Web y expone una **API HTTP** sencilla. Esto permite interactuar con WhatsApp de forma programática para enviar y recibir mensajes, siendo ideal para bots y notificaciones automáticas.

### ¿Qué es n8n?

**n8n** es una plataforma de automatización de flujos de trabajo (workflow automation) que permite conectar servicios mediante nodos (HTTP, bases de datos, APIs, etc.). En esta guía, se usa n8n para **recibir eventos (webhooks)** desde WAHA y responder automáticamente.

---

## 2. ✅ Requisitos Previos

* Servidor Linux (VPS o local) con **Docker** y **Docker Compose** instalados.
* **Puertos disponibles:** `3000` (para WAHA) y `5678` (para n8n).
* Un número de WhatsApp para vincular.

---

## 3. ⚙️ Preparación del Entorno

En tu servidor, crea el directorio del proyecto y un archivo de variables de entorno:

```bash
mkdir waha-n8n
cd waha-n8n

# Crear el archivo .env (opcional)
nano .env
````

**Contenido mínimo del `.env`:**

```ini
N8N_HOST=localhost
```

-----

## 4\. 📦 Docker Compose: Archivo Unificado

Crea el archivo `docker-compose.yaml` con la configuración completa para WAHA y n8n.

```bash
nano docker-compose.yaml
```

**Pega el contenido completo:**

```yaml
version: '3.8'

services:

  # Servicio WAHA - API de WhatsApp
  waha:
    image: devlikeapro/waha:latest
    container_name: waha
    restart: unless-stopped
    ports:
      - "3000:3000"  # Puerto público para WAHA
    environment:
      # Clave API para autenticar peticiones a WAHA (header X-Api-Key)
      WAHA_API_KEY: "mi_clave_secreta_2024"

      # Credenciales para el Dashboard web de WAHA
      WAHA_DASHBOARD_USERNAME: "admin"
      WAHA_DASHBOARD_PASSWORD: "password_seguro_123"
    
      # Credenciales para la documentación Swagger de la API
      WHATSAPP_SWAGGER_USERNAME: "swagger"
      WHATSAPP_SWAGGER_PASSWORD: "swagger_pass_456"
    volumes:
      # Persistencia de datos de las sesiones de WhatsApp (QR, login, etc.)
      - waha_data:/app/.waha
    networks:
      - automation-network
    
  # Servicio n8n - Automatización de flujos
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"  # Puerto público para la UI de n8n
    environment:
      # Zona horaria del servidor
      - TZ=America/Caracas

      # Configuración de host/puerto/protocolo para URLs internas de n8n
      - N8N_HOST=${N8N_HOST:-localhost}
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
    
      # URL base que usará n8n para generar URLs de webhooks
      - WEBHOOK_URL=http://${N8N_HOST:-localhost}:5678/
    volumes:
      # Persistencia de workflows, credenciales, etc.
      - n8n_data:/home/node/.n8n
    networks:
      - automation-network
    
volumes:
  waha_data:
    driver: local
  n8n_data:
    driver: local

networks:
  automation-network:
    driver: bridge
```

-----

## 5\. 🔬 Explicación de Componentes

| Componente | Uso |
| :--- | :--- |
| **`WAHA_API_KEY`** | Clave de seguridad obligatoria en el *header* `X-Api-Key` para usar la API de WAHA. |
| **`automation-network`** | Red interna de Docker que permite a `n8n` acceder a `waha` (y viceversa) usando sus nombres de servicio: `http://waha:3000`. |
| **`volumes`** | Permiten la persistencia de datos. `waha_data` guarda la sesión de WhatsApp, y `n8n_data` guarda tus workflows. |
| **`WEBHOOK_URL`** | Define la URL base que n8n usará para construir las direcciones de sus webhooks. |

-----

## 6\. 🛠️ Puesta en Marcha y Accesos

### Iniciar los Servicios

Desde el directorio `waha-n8n`, levanta los contenedores:

```bash
docker-compose up -d
```

### Accesos

Reemplaza `TU-IP` por la IP o dominio de tu servidor.

  * **WAHA Dashboard:** `http://TU-IP:3000` (Usar credenciales `WAHA_DASHBOARD_*`).
  * **n8n UI:** `http://TU-IP:5678` (La primera vez te pedirá crear un usuario).

### Vincular WhatsApp

1.  Accede a `http://TU-IP:3000`.
2.  Crea una nueva sesión (ej. `default`).
3.  Escanea el código QR con la app de WhatsApp (Dispositivos vinculados).
4.  Espera el estado **`WORKING`**.

-----

## 7\. 🕸️ Conexión Webhook: Recibir Mensajes (WAHA → n8n)

### Paso 7.1: Crear el Webhook en n8n

1.  En n8n, crea un **Nuevo Workflow**.
2.  Agrega el nodo **Webhook**.
3.  Configura **HTTP Method**: `POST` y **Path**: `whatsapp`.
4.  **Guarda y activa el Workflow**.

### Paso 7.2: Configurar WAHA para Enviar el Webhook

Desde la terminal, configura la sesión de WAHA para que envíe los mensajes entrantes al webhook interno de n8n:

```bash
curl -X POST http://TU-IP:3000/api/sessions/default/settings \
-H "X-Api-Key: mi_clave_secreta_2024" \
-H "Content-Type: application/json" \
-d '{
"webhooks": [
{
"url": "http://n8n:5678/webhook/whatsapp",
"events": ["message"]
}
]
}'
```

> **Nota:** La URL interna `http://n8n:5678` es clave. Usa la `WAHA_API_KEY` correcta.

-----

## 8\. 📤 Enviar Mensajes: Responder desde n8n (n8n → WAHA)

Para que n8n pueda responder a un mensaje recibido, o enviarlo de forma programada, debe hacer una llamada HTTP a la API de WAHA.

### Agregar el Nodo de Respuesta en n8n

Añade un nodo **HTTP Request** al workflow (después del Webhook y un posible nodo Code/Function para extraer datos):

  * **Method**: `POST`
  * **URL**: `http://waha:3000/api/sendText` (Endpoint de WAHA para enviar texto)
  * **Headers**:
      * `X-Api-Key`: `mi_clave_secreta_2024`
      * `Content-Type`: `application/json`
  * **Body (JSON)**:

<!-- end list -->

```json
{
  "session": "default",
  "chatId": "{{ $json.from }}", 
  "text": "¡Mensaje recibido! Te respondo automáticamente."
}
```

> Si el mensaje es una respuesta a un webhook, la expresión `{{ $json.from }}` debería contener el número de chat del remitente original.

-----

## 9\. 🔒 Nodo de WAHA en N8N

  * **Ir a**  n8n => Settings => Community nodes e instala:
  ```
  @devlikeapro/n8n-nodes-waha
  ```


<!-- end list -->