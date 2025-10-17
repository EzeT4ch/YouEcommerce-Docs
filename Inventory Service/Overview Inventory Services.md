# 📦 WMS (Warehouse Management System) – Detalle Funcional

## 🧭 Rol general
El WMS Service administra productos, inventario y ubicaciones.  
Es responsable de mantener la consistencia del stock y proporcionar información precisa sobre disponibilidad a otros módulos como Billing y el Orquestador del SaaS.

Debe funcionar tanto como:
- **Servicio interno** (para ecommerce)
- **API externa** (para otros sistemas de gestión de inventarios)

---

## ✅ Funcionalidades clave

| Categoría | Funcionalidad | Descripción |
|------------|----------------|--------------|
| Productos | CRUD de productos | Alta, modificación, eliminación y consulta de productos |
| Inventario | Control de stock | Registro y actualización de cantidades disponibles |
| Movimientos | Entradas, salidas, ajustes | Movimientos físicos o virtuales del stock |
| Ubicaciones | Control de almacenes, depósitos, zonas | Permite dividir inventario por lugar |
| Integración | API REST y eventos | Comunicación con Billing, Orquestador, etc. |
| Multi-tenant | Separación lógica por tenant | Cada cliente tiene su propio espacio de datos |
| Sincronización | Control de consistencia con Billing | Ajustes de stock automáticos tras pagos confirmados |

---

## 🧩 Entidades principales

### `Product`
| Campo | Tipo | Descripción |
|--------|------|-------------|
| `id` | GUID | Identificador único |
| `tenant_id` | GUID | Propietario del producto |
| `sku` | String | Código único del producto |
| `name` | String | Nombre del producto |
| `description` | String | Descripción opcional |
| `category` | String | Categoría o grupo |
| `price` | Decimal | Precio base de venta |
| `currency` | String | Moneda |
| `is_active` | Bool | Disponible o no |
| `created_at` | DateTime | Fecha de creación |

### `StockItem`
| Campo | Tipo | Descripción |
|--------|------|-------------|
| `id` | GUID | Identificador |
| `product_id` | GUID | FK a Product |
| `tenant_id` | GUID | Propietario |
| `location_id` | GUID | Almacén o depósito |
| `quantity_available` | Decimal | Cantidad disponible |
| `quantity_reserved` | Decimal | Cantidad comprometida |
| `updated_at` | DateTime | Última actualización |

### `StockMovement`
| Campo | Tipo | Descripción |
|--------|------|-------------|
| `id` | GUID | Identificador del movimiento |
| `product_id` | GUID | FK a Product |
| `tenant_id` | GUID | Tenant |
| `type` | Enum (`IN`, `OUT`, `ADJUSTMENT`, `RESERVATION`) | Tipo de movimiento |
| `quantity` | Decimal | Cantidad |
| `reason` | String | Motivo o nota |
| `reference_id` | String | ID externo (orden, factura, etc.) |
| `created_at` | DateTime | Fecha del movimiento |

### `Location`
| Campo | Tipo | Descripción |
|--------|------|-------------|
| `id` | GUID | Identificador |
| `tenant_id` | GUID | Tenant |
| `name` | String | Nombre del almacén o zona |
| `type` | Enum (`WAREHOUSE`, `STORE`, `VIRTUAL`) | Tipo de ubicación |
| `is_active` | Bool | Disponible para stock o no |

---

## 🔄 Flujos operativos

### 1. **Alta de producto**
- Se crea producto con `POST /wms/products`.
- Si `auto_stock_enabled=true` (flag), se inicializa con stock `0` en `default location`.

### 2. **Entrada de stock**
- Ingreso de unidades (`POST /wms/movements`) con `type=IN`.
- Actualiza `quantity_available` del `StockItem`.

### 3. **Salida de stock (venta o reserva)**
- Cuando Billing confirma una orden pagada, envía un `StockReservation` o `StockDeduction`.
- El WMS actualiza `quantity_reserved` o `quantity_available`.

### 4. **Ajuste de inventario**
- Manual o automático (`type=ADJUSTMENT`).
- Se deja registro del motivo.

### 5. **Sincronización con Billing**
- Eventual o inmediata (según flag).
- A la confirmación de pago, Billing → `POST /wms/reserve` o evento `OrderPaid`.
- WMS descuenta stock.

---

## ⚙️ Reglas de negocio

1. **No se puede eliminar un producto con stock positivo.**
2. **Los movimientos deben ser atómicos** (una transacción = un cambio de stock).
3. **Los valores de stock no pueden ser negativos** (salvo flag explícito).
4. **Cada movimiento debe generar log de auditoría.**
5. **Cada tenant tiene su propio conjunto de productos, ubicaciones y stock.**
6. **Ajustes manuales requieren `reason` obligatorio.**

---

## 🌐 Endpoints REST

| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/health` | Health check |
| **Productos** |||
| GET | `/wms/products` | Listar productos |
| GET | `/wms/products/{id}` | Obtener detalle de producto |
| POST | `/wms/products` | Crear nuevo producto |
| PUT | `/wms/products/{id}` | Actualizar producto |
| DELETE | `/wms/products/{id}` | Eliminar producto |
| **Inventario y stock** |||
| GET | `/wms/stock` | Obtener stock general |
| GET | `/wms/stock/{productId}` | Stock de producto específico |
| POST | `/wms/stock/movements` | Registrar movimiento (`IN`, `OUT`, etc.) |
| GET | `/wms/stock/movements` | Listar movimientos por fecha o producto |
| **Ubicaciones** |||
| GET | `/wms/locations` | Listar almacenes |
| POST | `/wms/locations` | Crear almacén o zona |
| **Integraciones** |||
| POST | `/wms/reserve` | Reservar stock por orden externa |
| POST | `/wms/release` | Liberar stock reservado |
| POST | `/wms/sync` | Sincronización completa (Billing/Orquestador) |

---

## 🧪 Feature Flags (WMS)

| Flag | Descripción | Alcance |
|------|--------------|---------|
| `wms.auto_stock_enabled` | Crea stock inicial al crear producto | Tenant |
| `wms.allow_negative_stock` | Permite stock negativo | Global |
| `wms.multi_location_enabled` | Habilita múltiples almacenes por tenant | Tenant |
| `wms.reservation_mode` | Reserva stock antes de confirmación de pago | Tenant |
| `wms.audit_log_enabled` | Guarda todos los movimientos | Global |
| `wms.sync_billing_enabled` | Habilita sincronización automática con Billing | Global |

---

## 🔔 Integraciones

| Servicio | Tipo | Descripción |
|-----------|------|--------------|
| **Auth** | JWT | Validación de usuario y tenant |
| **Billing** | REST/Eventos | Reserva o descuento de stock post-pago |
| **Notifications** | Evento | Alerta de bajo stock o ajuste manual |
| **Orquestador SaaS** | REST | Usa endpoints de productos y stock |

---

## 📄 Esquema mínimo de tablas

- `Products(id, tenant_id, sku, name, description, category, price, currency, is_active, created_at)`
- `StockItems(id, product_id, tenant_id, location_id, quantity_available, quantity_reserved, updated_at)`
- `StockMovements(id, product_id, tenant_id, type, quantity, reason, reference_id, created_at)`
- `Locations(id, tenant_id, name, type, is_active)`

---

## 🧭 Recomendaciones de diseño

- Mantener el módulo **autónomo y transaccional**.
- Usar **CQRS** si el volumen de operaciones crece.
- Incluir **métricas y alertas** para niveles bajos de stock.
- Preparar estructura para **event-driven** en el futuro.
- Mantener auditoría completa de todos los movimientos (alta prioridad).

---

## 🧾 Ejemplo de flujo de integración

**Orden pagada → descuento de stock**
```json
POST /wms/reserve
{
  "order_id": "ORD-2301",
  "tenant_id": "TEN-002",
  "items": [
    { "product_id": "SKU-1001", "quantity": 2 },
    { "product_id": "SKU-1002", "quantity": 1 }
  ]
}
