# 💳 Payment Adapter / Gateway Layer – Detalle Funcional

## 🧭 Rol general
Servicio que centraliza la integración con múltiples pasarelas de pago.  
Provee una API unificada para iniciar, confirmar y cancelar pagos sin depender de un proveedor específico.

Este módulo **no maneja lógica de negocio de facturación o pedidos** (eso lo hace Billing), sino que actúa como **broker** entre Billing y los proveedores de pago externos.

---

## ✅ Funcionalidades clave

| Categoría | Funcionalidad | Descripción |
|------------|----------------|--------------|
| Inicialización | Crear intención de pago | Generar sesión de pago con proveedor externo |
| Confirmación | Validar resultado del pago | Recepción de callback o webhook de proveedor |
| Cancelación | Cancelar un pago pendiente | Solicitud desde Billing o usuario |
| Adaptadores | Integración modular | Conectores individuales para cada proveedor |
| Webhooks | Recepción de notificaciones externas | Procesamiento seguro de actualizaciones |
| Firma / seguridad | Verificación de integridad de eventos | Previene fraudes y falsos positivos |
| Feature flags | Control de proveedores habilitados | Por tenant o entorno |
| Sandbox | Simulación de pagos | Para testear sin costos |

---

## 🧩 Entidades principales

### `PaymentIntent`
| Campo | Tipo | Descripción |
|--------|------|-------------|
| `id` | GUID | Identificador único interno |
| `tenant_id` | GUID | Tenant asociado |
| `order_id` | GUID | ID del pedido en Billing |
| `provider` | Enum (`STRIPE`, `MERCADOPAGO`, `PAYPAL`, `MOCK`) | Proveedor de pago |
| `status` | Enum (`CREATED`, `PENDING`, `PAID`, `FAILED`, `CANCELLED`) | Estado actual |
| `amount` | Decimal | Monto total |
| `currency` | String | Moneda |
| `provider_reference` | String | ID asignado por el proveedor |
| `return_url` | String | URL de redirección tras pago |
| `created_at` | DateTime | Creación |
| `updated_at` | DateTime | Última actualización |

### `ProviderConfig`
| Campo | Tipo | Descripción |
|--------|------|-------------|
| `id` | GUID | Identificador |
| `tenant_id` | GUID | Tenant |
| `provider` | Enum | Nombre del proveedor |
| `api_key` | String | Clave privada |
| `public_key` | String | Clave pública (si aplica) |
| `webhook_secret` | String | Clave de validación de eventos |
| `enabled` | Bool | Estado |
| `sandbox_mode` | Bool | Entorno de prueba habilitado |

### `PaymentEvent`
| Campo | Tipo | Descripción |
|--------|------|-------------|
| `id` | GUID | Identificador |
| `tenant_id` | GUID | Tenant |
| `provider` | String | Proveedor |
| `event_type` | String | Ej: `payment.succeeded`, `payment.failed` |
| `payload` | JSON | Datos originales del proveedor |
| `signature` | String | Firma HMAC del evento |
| `received_at` | DateTime | Fecha de recepción |

---

## 🔄 Flujos principales

### 1. **Inicio de pago**
- Billing llama a `/payments/initiate` con `order_id`, `total`, `currency`.
- Payment Adapter:
  - Identifica proveedor habilitado del tenant.
  - Crea `PaymentIntent`.
  - Llama a API externa del proveedor.
  - Devuelve URL de checkout o credenciales para el frontend.

### 2. **Pago confirmado**
- El proveedor (ej. Stripe, MercadoPago) llama a `/payments/webhook/{provider}`.
- Payment Adapter valida firma (`webhook_secret`).
- Actualiza `PaymentIntent` a `PAID`.
- Envía callback a Billing (`/billing/orders/{id}` → PATCH `status=paid`).

### 3. **Cancelación o fallo**
- En caso de error o cancelación del usuario:
  - Se actualiza estado a `CANCELLED` o `FAILED`.
  - Se notifica al Orquestador / frontend vía webhook interno.

### 4. **Modo sandbox**
- Si el tenant tiene `sandbox_mode=true`, no se contacta al proveedor real.
- Se simula respuesta inmediata (`PAID` o `FAILED`) según configuración.

---

## ⚙️ Reglas de negocio

1. **Un `PaymentIntent` pertenece a un solo `Order`.**
2. **No se puede crear más de un pago activo por orden.**
3. **Cada webhook debe validar firma HMAC del proveedor.**
4. **Los pagos en sandbox nunca afectan órdenes reales.**
5. **Cada tenant puede configurar sus proveedores habilitados.**
6. **El estado `PAID` es el único que genera callback a Billing.**

---

## 🌐 Endpoints REST

| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/health` | Health check |
| **Pagos** |||
| POST | `/payments/initiate` | Crear pago con proveedor configurado |
| GET | `/payments/{id}` | Consultar estado de pago |
| PATCH | `/payments/{id}/cancel` | Cancelar pago pendiente |
| **Webhooks externos** |||
| POST | `/payments/webhook/{provider}` | Recepción de evento externo (Stripe, MP, etc.) |
| **Configuraciones** |||
| GET | `/providers` | Listar proveedores disponibles |
| GET | `/providers/{tenant_id}` | Obtener configuraciones por tenant |
| POST | `/providers/{tenant_id}` | Guardar configuración de proveedor |
| PUT | `/providers/{tenant_id}/{provider}` | Actualizar configuración |
| **Sandbox / testing** |||
| POST | `/sandbox/payments/simulate` | Simular pago (sandbox mode) |

---

## 🧪 Feature Flags (Payment Adapter)

| Flag | Descripción | Alcance |
|------|--------------|---------|
| `payments.sandbox_mode` | Simula pagos (sin proveedor real) | Tenant |
| `payments.multiple_providers` | Permite más de un proveedor activo | Tenant |
| `payments.retry_webhooks` | Reintenta envío de callbacks a Billing | Global |
| `payments.log_payloads` | Guarda payloads completos para debugging | Global |
| `payments.enable_stripe` | Activa integración Stripe | Global |
| `payments.enable_mercadopago` | Activa integración MercadoPago | Global |
| `payments.enable_paypal` | Activa integración PayPal | Global |

---

## 🔔 Integraciones

| Servicio | Interacción | Descripción |
|-----------|-------------|--------------|
| **Billing** | REST / callback | Confirmación de pago (`PATCH /billing/orders/{id}`) |
| **Auth** | JWT | Validación de tenant |
| **Notification Service** | Evento `payment.success` / `payment.failed` | Email o webhook |
| **Orquestador SaaS** | REST | Inicia pagos desde el frontend |
| **WMS** | Indirecta (vía Billing) | Reserva de stock tras pago exitoso |

---

## 🧭 Ejemplo de flujo completo

1. Billing crea orden `ORD-2301`.
2. Llama a `POST /payments/initiate`:
   ```json
   {
     "order_id": "ORD-2301",
     "tenant_id": "TEN-002",
     "amount": 13310,
     "currency": "ARS",
     "return_url": "https://tienda.com/checkout/success"
   }
```

1. Payment Adapter crea sesión en MercadoPago y responde:
    
    ```JSON
    {   
	    "payment_id": "PAY-9001",   
	    "provider": "MERCADOPAGO",   
	    "checkout_url": "https://mercadopago.com/initiate/ABC123" 
	}
    ```
    
2. Usuario paga → MP envía webhook:
    
    ```JSON
    {   
    "event": "payment.succeeded",   
    "data": 
	    { 
		    "payment_id": "PAY-9001", 
		    "order_id": "ORD-2301", 
		    "amount": 13310 
		} 
	}
    ```
    
3. Payment Adapter valida firma y ejecuta:
    
    ```JSON
    PATCH /billing/orders/ORD-2301 
    { 
	    "status": "paid" 
	}
    ```
    
1. Billing emite factura → Notification Service envía email.

---

## 📄 Esquema mínimo de tablas

- `PaymentIntents(id, tenant_id, order_id, provider, status, amount, currency, provider_reference, return_url, created_at, updated_at)`
    
- `ProviderConfigs(id, tenant_id, provider, api_key, public_key, webhook_secret, enabled, sandbox_mode)`
    
- `PaymentEvents(id, tenant_id, provider, event_type, payload, signature, received_at)`
    

---

## 🧭 Recomendaciones de diseño

- Aplicar **patrón Strategy** para adaptadores de pago (`IPaymentProvider`).
- Asegurar **idempotencia** de los webhooks.
- Diseñar los callbacks a Billing con **reintentos automáticos**.
- Soportar **versionado de APIs de proveedores**.
- Registrar todos los eventos de entrada/salida para auditoría.
- Usar **sandbox_mode** activado por defecto en entornos de desarrollo.

---

## 🧾 Ejemplo de respuesta de API

```JSON
{   
	"payment_id": "PAY-9001",   
	"provider": "STRIPE",   
	"status": "PENDING",   
	"checkout_url": "https://checkout.stripe.com/pay/abc123",
	"tenant_id": "TEN-002",   
	"created_at": "2025-10-17T17:45:00Z" 
}
```