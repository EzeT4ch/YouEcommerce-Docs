# 🛍️ Storefront – Detalle Funcional (Angular + PrimeNG + Material)

## 🧭 Rol general
El **Storefront** es la aplicación pública orientada al usuario final.  
Permite navegar el catálogo, realizar compras, registrar usuarios y completar pagos integrados.  
Soporta **multi-tenant** mediante configuración dinámica por dominio o subdominio.

---

## 🎯 Objetivos

- UI liviana, responsive y personalizable.
- Integración total con:
  - WMS (productos)
  - Billing (órdenes)
  - Payment Adapter (checkout)
  - Notification Service (confirmaciones)
- Branding y configuración dinámica por tenant.

---

## ✅ Requisitos funcionales

| Categoría | Funcionalidad | Descripción | Prioridad |
|------------|----------------|--------------|------------|
| Catálogo | Listado y búsqueda | Visualización paginada de productos | Must-have |
| Detalle de producto | Ficha visual con fotos y precios | Integrado con WMS | Must-have |
| Carrito | Agregar/quitar productos | Persistencia local o en backend | Must-have |
| Checkout | Flujo de pago | Billing → Payment Adapter | Must-have |
| Autenticación | Registro/login del comprador | Auth Service | Must-have |
| Seguimiento | Ver estado de pedido | Billing Service | Should-have |
| Personalización | Tema y branding | Configuración por tenant | Could-have |

---

## 🧩 Arquitectura

src/  
├── app/  
│ ├── core/ # Servicios globales  
│ ├── shared/ # UI Components comunes  
│ ├── modules/  
│ │ ├── catalog/  
│ │ ├── cart/  
│ │ ├── checkout/  
│ │ ├── orders/  
│ │ ├── auth/  
│ │ └── tenant/  
│ ├── layouts/ # Header/Footer/Sidebar  
│ ├── app-routing.module.ts  
│ └── app.module.ts  
├── assets/  
│ ├── themes/  
│ └── images/  
├── environments/  
│ └── environment.ts


---

## ⚙️ Dependencias principales

| Paquete | Uso |
|----------|-----|
| `@angular/material` | UI consistente |
| `primeng` | Listados, modales, formularios |
| `ngx-translate/core` | Internacionalización |
| `@ngxs/store` | Manejo de estado global |
| `ngx-cart` (custom) | Lógica de carrito compartida |
| `ngx-feature-flag` | Control de flags por tenant |
| `jwt-decode` | Autenticación |
| `ngx-spinner` | Feedback visual |
| `rxjs` | Observables para estados |

---

## 🧠 Patrones clave

- **Reactive Forms** para validación en checkout.  
- **Guards** para proteger rutas (`AuthGuard`, `CheckoutGuard`).  
- **Interceptors** para inyectar JWT y manejar errores globales.  
- **Lazy loading** para performance.  
- **Dynamic tenant configuration** cargada al inicio (por dominio).

---

## 🔄 Flujos principales

### 1. **Catálogo**
- GET `/wms/products?tenant_id=xyz`
- Renderizado en grilla (`p-card`, `p-image`).

### 2. **Carrito**
- Items persistidos localmente (`localStorage`).
- `CartService` mantiene estado global.

### 3. **Checkout**
1. Validación del usuario (Auth).
2. POST `/billing/orders` → crea orden.
3. POST `/payments/initiate` → genera sesión.
4. Redirección a pasarela.
5. Webhook del pago → `PATCH /billing/orders/{id}` (`paid`).
6. Redirección a página de confirmación.

---

## 🧪 Feature Flags (Storefront)

| Flag | Descripción | Alcance |
|------|--------------|---------|
| `storefront.guest_checkout` | Permite comprar sin login | Tenant |
| `storefront.multi_tenant` | Activa modo multi-tenant | Global |
| `storefront.dynamic_theme` | Tema visual según tenant | Tenant |
| `storefront.discount_banner` | Muestra banners de promo | Tenant |
| `storefront.experimental_checkout` | Nuevo flujo de pago | Global |
| `storefront.websockets_enabled` | Estado de orden en tiempo real | Global |

---

## 🎨 UI / UX Guidelines

- **Frameworks:** PrimeNG + Angular Material  
- **Componentes base:**
  - `p-card` para productos.
  - `mat-toolbar` + `mat-sidenav` para layout.
  - `p-toast` para feedback.
- **Responsive:** PrimeFlex grid system (`p-col-12`, `p-md-6`, etc.)
- **Temas:** se cargan dinámicamente (`assets/themes/{tenant}.scss`).
- **Dark mode:** configurable por flag.
- **Accesibilidad (a11y):** cumplir WCAG 2.1 AA.

---

## 🧭 Recomendaciones finales

- Implementar `TenantResolver` que cargue configuración antes del bootstrap.
- Preparar PWA (instalable).
- Integrar métricas (Google Analytics / custom).
- Testing E2E con Cypress para checkout.
- Cuidar rendimiento (lazy load, prefetching de imágenes, caching).

---
