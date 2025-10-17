# 📩 Notification Service – Detalle Funcional

## 🧭 Rol general
Servicio responsable de la gestión y envío de notificaciones del ecosistema.  
Su propósito es centralizar la mensajería (email, webhook, push u otros canales), garantizando:
- Consistencia en la entrega de mensajes.
- Desacoplamiento entre módulos emisores (Billing, WMS, Auth, etc.).
- Extensibilidad para nuevos canales y plantillas.

---

## ✅ Funcionalidades clave

| Categoría | Funcionalidad | Descripción |
|------------|----------------|--------------|
| Envío básico | Envío de email transaccional | Mensajes automáticos de eventos del sistema |
| Webhooks | Notificación HTTP a sistemas externos | Ideal para integraciones o SaaS externos |
| Plantillas | Administración de templates parametrizables | Personalización de mensajes |
| Log de envíos | Registro del historial de notificaciones | Auditoría y debugging |
| Retries | Reintentos automáticos de notificaciones fallidas | Robustez ante errores externos |
| Feature Flags | Activación o desactivación de canales | Control granular por tenant |
| Multi-tenant | Separación de configuraciones y plantillas por cliente | Aislamiento y personalización |

---

## 🧩 Entidades principales

### `Notification`
| Campo | Tipo | Descripción |
|--------|------|-------------|
| `id` | GUID | Identificador |
| `tenant_id` | GUID | Tenant propietario |
| `type` | Enum (`EMAIL`, `WEBHOOK`, `PUSH`) | Tipo de canal |
| `status` | Enum (`PENDING`, `SENT`, `FAILED`, `RETRYING`) | Estado actual |
| `template_id` | GUID | Plantilla utilizada |
| `recipient` | String | Destinatario (email o URL) |
| `payload` | JSON | Datos del mensaje (dinámicos) |
| `error_message` | String | Último error (si aplica) |
| `created_at` | DateTime | Fecha de creación |
| `sent_at` | DateTime | Fecha de envío (si aplica) |

### `Template`
| Campo | Tipo | Descripción |
|--------|------|-------------|
| `id` | GUID | Identificador |
| `tenant_id` | GUID | Tenant propietario |
| `name` | String | Nombre legible |
| `channel` | Enum (`EMAIL`, `WEBHOOK`) | Canal de uso |
| `subject` | String | Asunto (solo para email) |
| `body` | Text | Contenido con variables (`{{user_name}}`, `{{order_id}}`) |
| `created_at` | DateTime | Fecha de creación |

### `WebhookConfig`
| Campo | Tipo | Descripción |
|--------|------|-------------|
| `id` | GUID | Identificador |
| `tenant_id` | GUID | Tenant propietario |
| `target_url` | String | URL del webhook receptor |
| `secret` | String | Clave para firma HMAC |
| `events_subscribed` | JSON | Lista de eventos que escucha (`order.paid`, etc.) |
| `is_active` | Bool | Estado del webhook |

---

## 🔄 Flujos de notificación

### 1. **Evento → Mensaje**
- Un servicio (Billing, WMS, Auth) genera un evento (`order.paid`, `user.registered`, etc.).
- Se publica por REST o cola a Notification Service.
- Notification selecciona la plantilla adecuada y canal.
- Renderiza el mensaje reemplazando variables dinámicas.

### 2. **Envío**
- Si `EMAIL`: usa adaptador SMTP o proveedor externo (SendGrid, Mailgun, etc.).
- Si `WEBHOOK`: realiza `POST` firmado (HMAC) al target.
- Si `PUSH`: (futuro) usa adaptador FCM o equivalente.
- Actualiza estado (`PENDING` → `SENT` o `FAILED`).

### 3. **Reintento**
- Si el envío falla, entra a cola de reintento (`RETRYING`).
- Máximo de reintentos configurable por flag o tenant.

### 4. **Auditoría**
- Todos los eventos se registran en la tabla `Notifications` con timestamps.
- Logs centralizados por tenant y por canal.

---

## ⚙️ Reglas de negocio

1. Toda notificación debe pertenecer a un tenant.
2. Los templates se resuelven por tenant; si no existe, se usa uno global por defecto.
3. Un fallo en un canal no afecta los demás.
4. Cada webhook debe ser firmado (`HMAC-SHA256`) si tiene `secret` configurado.
5. Las notificaciones fallidas deben tener visibilidad en logs y panel admin.
6. El servicio debe ser idempotente (si se reenvía un mismo `notification_id`, no se duplica).

---

## 🌐 Endpoints REST

| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/health` | Health check |
| **Notificaciones** |||
| POST | `/notifications` | Crear y enviar notificación |
| GET | `/notifications` | Listar notificaciones |
| GET | `/notifications/{id}` | Obtener detalle |
| **Templates** |||
| GET | `/templates` | Listar plantillas |
| GET | `/templates/{id}` | Obtener detalle |
| POST | `/templates` | Crear plantilla |
| PUT | `/templates/{id}` | Actualizar plantilla |
| DELETE | `/templates/{id}` | Eliminar plantilla |
| **Webhooks** |||
| GET | `/webhooks` | Listar webhooks |
| POST | `/webhooks` | Crear webhook |
| PUT | `/webhooks/{id}` | Actualizar configuración |
| DELETE | `/webhooks/{id}` | Eliminar webhook |
| POST | `/webhooks/test` | Probar envío manual |
| **Eventos simulados (sandbox)** |||
| POST | `/sandbox/event` | Simular evento `order.paid`, `user.registered`, etc. |

---

## 🧪 Feature Flags (Notification)

| Flag | Descripción | Alcance |
|------|--------------|---------|
| `notifications.email_enabled` | Habilita canal de correo electrónico | Tenant |
| `notifications.webhooks_enabled` | Permite envío de webhooks externos | Tenant |
| `notifications.push_enabled` | Activa canal push (futuro) | Global |
| `notifications.retry_enabled` | Reintentos automáticos ante error | Global |
| `notifications.audit_enabled` | Registra logs detallados por envío | Global |
| `notifications.sandbox_mode` | Modo prueba, no envía mensajes reales | Tenant |

---

## 🔔 Integraciones

| Servicio | Interacción | Descripción |
|-----------|-------------|--------------|
| **Billing** | Evento `order.paid` → email / webhook | Notifica pago confirmado |
| **Auth** | Evento `user.registered` | Envía email de bienvenida |
| **WMS** | Evento `stock.low` | Alerta de bajo inventario |
| **Orquestador** | REST o webhook | Envío manual de notificaciones desde panel admin |

---

## 🛠️ Tecnología recomendada

- `.NET 9` + ASP.NET Core (Web API)
- `SQL Server` (`Notifications`, `Templates`, `Webhooks`)
- `BackgroundService` (cola interna con reintentos)
- `MailKit` o `SendGrid SDK` para emails
- `HttpClientFactory` para webhooks
- `Serilog` + `Application Insights` para logs
- `Unleash` o `Azure App Configuration` para feature flags

---

## 🧭 Recomendaciones de diseño

- Diseñar canales como **estrategias intercambiables** (`INotificationChannel`).
- Mantener lógica de envío desacoplada del evento original.
- Preparar integración futura con **bus de mensajes (RabbitMQ, NATS)**.
- Agregar **endpoint de test** (`/webhooks/test`) para validación rápida de integradores.
- Implementar **limpieza automática de logs antiguos** para optimizar espacio.

---

## 🧾 Ejemplo de notificación (email)

```json
POST /notifications
{
  "type": "EMAIL",
  "tenant_id": "TEN-002",
  "template_id": "TPL-ORDER-PAID",
  "recipient": "cliente@ejemplo.com",
  "payload": {
    "user_name": "Juan Pérez",
    "order_id": "ORD-2301",
    "total": 13310
  }
}
```

**Respuesta**

```JSON
{   
	"id": "NOTIF-789",   
	"status": "SENT",   
	"sent_at": "2025-10-17T18:12:00Z" 
}
```

## 🧾 Ejemplo de webhook firmado

**Request**

```JSON
POST /external/webhook HTTP/1.1 Content-Type: application/json X-Signature: sha256=2d1f6c77...  
{   
	"event": "order.paid",   
	"data": {     
		"order_id": "ORD-2301",     
		"tenant_id": "TEN-002",     
		"total": 13310   
	} 
}
```

**Firma (pseudo)**

`signature = HMAC_SHA256(secret, payload)`

---
	
## 📄 Esquema mínimo de tablas

- `Notifications(id, tenant_id, type, status, template_id, recipient, payload, error_message, created_at, sent_at)`
    
- `Templates(id, tenant_id, name, channel, subject, body, created_at)`
    
- `Webhooks(id, tenant_id, target_url, secret, events_subscribed, is_active)`
    

---

## 🧪 Métricas recomendadas

- Envíos por hora / canal.
- Ratio de fallos por canal.
- Tiempo promedio de entrega (webhook + email).
- Cantidad de reintentos ejecutados.
- Tenants con más volumen de mensajes.

---

## 📬 Ejemplo de evento recibido desde Billing

```JSON
{   
	"event": "order.paid",   
	"tenant_id": "TEN-002",   
	"data": {     
		"order_id": "ORD-2301",     
		"total": 13310,     
		"currency": "ARS",     
		"email": "cliente@ejemplo.com"   
	} 
}
```

El Notification Service determina:
- Canal (email / webhook / push),
- Plantilla (`order.paid`),
- Mensaje final y entrega.