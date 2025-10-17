# 📦 Billing Service – Detalle Funcional (Versión Extendida)

## 🧭 Rol general
Módulo encargado de la gestión de órdenes, cálculo de totales, generación de comprobantes y control de estados de facturación.  
Opera como un servicio autónomo que puede ser invocado por el orquestador del SaaS o por terceros vía API.

Su diseño debe soportar:
- **Facturación directa** (pago inmediato)
- **Facturación diferida** (pedido → pago → emisión)
- **Integración con múltiples fuentes** (ecommerce, POS, app móvil)
- **Multi-tenant** y **multi-moneda**

---

## ✅ Funcionalidades clave

| Categoría              | Funcionalidad                                                    | Descripción resumida |
|------------------------|-----------------------------------------------------------------|----------------------|
| Órdenes               | Creación, consulta, actualización de estado                      | Maneja pedidos y su estado de ciclo de vida |
| Cálculo financiero     | Subtotal, descuentos, impuestos, envío                           | Lógica de negocio de precios |
| Comprobantes           | Generación de factura / ticket                                   | Mock inicial, extensible a integración fiscal |
| Estados y transición   | Control de estados del pedido                                   | Reglas estrictas de cambio de estado |
| Multi-tenant / multi-moneda | Soporte para varios clientes / monedas                      | `tenant_id` y `currency` incluidos en todo registro |
| Integración            | Interfaz REST + eventos (a futuro)                              | Se conecta con orquestador y notificaciones |
| Feature flags          | Activar módulos o comportamientos por tenant                    | Ej. facturación real vs simulada |

---

### 🌐 Endpoints (iniciales)
| Método | Ruta                                 | Descripción |
|--------|--------------------------------------|-------------|
| GET    | `/health`                             | Health check |
| POST   | `/billing/orders`                     | Crear orden (`pending`) |
| GET    | `/billing/orders`                     | Listar órdenes del tenant |
| GET    | `/billing/orders/{id}`                | Detalle de una orden |
| PATCH  | `/billing/orders/{id}`                | Actualizar estado (`pending`→`paid`, `paid`→`refunded`, etc.) |
| POST   | `/billing/orders/{id}/invoice`        | Generar comprobante (mock/real) |
| GET    | `/billing/invoices/{id}`              | Obtener comprobante (detalle/PDF) |
| GET    | `/billing/config/taxes`               | Obtener configuración de impuestos (por tenant) |
| PUT    | `/billing/config/taxes`               | Actualizar configuración de impuestos |
| GET    | `/billing/config/discounts`           | Obtener reglas de descuentos |
| PUT    | `/billing/config/discounts`           | Actualizar reglas de descuentos |
| POST   | `/billing/orders/{id}/recalculate`    | Forzar recálculo de totales |
| POST   | `/billing/sandbox/pay/{orderId}`      | **Sandbox**: simular pago exitoso/cancelado |

**Notas**  
- Todos los endpoints (salvo `health`) requieren JWT válido con `tenant_id`.
- `recalculate` es útil para auditorías y consistencia.

---

### 🧪 Feature Flags (Billing)
| Flag                          | Propósito | Alcance |
|------------------------------|-----------|---------|
| `billing.real_enabled`       | Activa integración fiscal real (vs mock) | Tenant |
| `billing.mock_invoice_pdf`   | Genera PDF simulado para comprobantes | Tenant |
| `billing.currency_multi`     | Habilita multi-moneda | Global/Tenant |
| `billing.auto_invoice`       | Emite comprobante automáticamente al marcar `paid` | Tenant |
| `billing.allow_refunds`      | Permite refund de órdenes pagadas | Tenant |
| `billing.sandbox_mode`       | Expone endpoints de simulación de pago | Tenant |
| `billing.strict_status_flow` | Enforce de transiciones válidas de estado | Global |


---

## 🧩 Entidades principales

### `Order`
Representa una transacción de compra, puede estar pendiente, pagada o cancelada.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | GUID | Identificador único |
| `tenant_id` | GUID | Propietario del pedido |
| `user_id` | GUID | Cliente asociado (opcional) |
| `status` | Enum (`pending`, `paid`, `cancelled`, `refunded`) | Estado del pedido |
| `subtotal` | Decimal | Total antes de impuestos |
| `tax` | Decimal | Impuestos calculados |
| `discount` | Decimal | Descuento aplicado |
| `total` | Decimal | Total final |
| `currency` | String | Moneda del pedido |
| `items` | List\<OrderItem\> | Ítems de la orden |
| `created_at` | DateTime | Fecha de creación |
| `updated_at` | DateTime | Última modificación |

### `OrderItem`
Detalle de productos o servicios incluidos en la orden.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `product_id` | String | Identificador externo |
| `name` | String | Nombre del producto |
| `quantity` | Int | Cantidad |
| `unit_price` | Decimal | Precio unitario |
| `tax_rate` | Decimal | Porcentaje de impuesto aplicado |
| `total_price` | Decimal | Total del ítem |

### `Invoice` *(fase 2 o posterior)*
Representa la emisión del comprobante asociado a una orden.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | GUID | Identificador |
| `order_id` | GUID | FK hacia Order |
| `invoice_number` | String | Número de factura o recibo |
| `issued_at` | DateTime | Fecha de emisión |
| `external_ref` | String | ID en sistema fiscal externo (mock al inicio) |
| `pdf_url` | String | URL o path del PDF generado |

---

## 🔄 Flujo general de facturación

### 1. **Creación del pedido**
- El orquestador (frontend o API) envía el carrito.
- Billing valida estructura y precios (consulta o recibe de WMS).
- Crea registro en estado `pending`.

### 2. **Confirmación de pago**
- El Payment Adapter notifica a Billing (`PATCH /billing/orders/{id}` → `status=paid`).
- Billing recalcula totales para consistencia.
- Si todo es correcto, genera evento interno o invoca `Notification Service`.

### 3. **Emisión de comprobante (fase 2)**
- Si el tenant tiene flag `billing.real_enabled=true`, se genera comprobante real.
- Caso contrario, se emite factura mock (PDF local o JSON con número simulado).

### 4. **Estados válidos y transiciones**

| Estado actual | Acción | Nuevo estado | Condición |
|----------------|--------|---------------|------------|
| `pending` | Pago confirmado | `paid` | Pago exitoso |
| `pending` | Cancelar pedido | `cancelled` | Por cliente o timeout |
| `paid` | Reembolso | `refunded` | Solo admins |
| `cancelled` | Ninguna | — | Inmutable |
| `refunded` | Ninguna | — | Inmutable |

---

## ⚙️ Reglas de negocio

1. **Los montos no se confían al cliente:** siempre se recalculan.
2. **Toda orden pertenece a un tenant.**
3. **No se puede emitir factura sin estado `paid`.**
4. **Descuentos e impuestos deben definirse por política o configuración.**
5. **Un pedido solo puede tener una factura activa.**
6. **Soporte para `sandbox mode` por tenant** (ideal para test de integración).
7. **Transiciones válidas** :  
   - `pending` → `paid` (pago OK)  
   - `pending` → `cancelled` (timeout/usuario)  
   - `paid` → `refunded` (admin)  

---
### 🔔 Integraciones

- **Auth**: validar JWT y extraer `tenant_id`.
- **Payment Adapter**: webhooks o llamadas que disparan `paid`.
- **Notifications**: envío de comprobante y estado de la orden.
- **WMS** (futuro): descuento de stock al confirmar pago.

---
## 🧮 Cálculo de totales

Totales se calculan de forma determinística en backend:

```csharp
subtotal = Σ (quantity × unit_price)
discount = subtotal × discount_rate
tax = (subtotal - discount) × tax_rate
total = subtotal - discount + tax
```


## 🧾 Ejemplo de respuesta de orden pagada


```{   "id": "ORD-2301",   "status": "paid",   "tenant_id": "TEN-002",   "subtotal": 11000,   "tax": 2310,   "discount": 0,   "total": 13310,   "currency": "ARS",   "items": [     { "product_id": "SKU-1001", "name": "Zapato", "quantity": 2, "unit_price": 5000 },     { "product_id": "SKU-1002", "name": "Medias", "quantity": 1, "unit_price": 1000 }   ],   "invoice": {     "invoice_number": "A-0001-00001234",     "issued_at": "2025-10-17T16:00:00Z",     "pdf_url": "https://cdn.saas.com/invoices/ORD-2301.pdf"   } }`