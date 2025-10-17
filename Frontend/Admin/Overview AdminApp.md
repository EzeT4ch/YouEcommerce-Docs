# 🧩 Frontend Admin – Detalle Funcional (Angular + PrimeNG + Material)

## 🧭 Rol general
El **Frontend Admin** es la aplicación web utilizada por los dueños de tiendas (tenants) y operadores internos para gestionar todo el ciclo del negocio:  
productos, stock, usuarios, facturación, pagos, notificaciones y configuración.

Se implementará como una **Single Page Application (SPA)** en **Angular 18+**, utilizando **PrimeNG** y **Angular Material** como librerías base de componentes visuales.

---

## 🎯 Objetivos principales

- Ofrecer una **UI moderna, modular y rápida**, optimizada para desktop y tablets.
- Reutilizar componentes para distintos tenants (branding adaptable).
- Integrarse completamente con los microservicios:
  - Auth
  - Billing
  - WMS (Inventario)
  - Notifications
  - Payment Adapter
- Soportar **Feature Flags** para activar módulos según plan del cliente (Free / Pro / Enterprise).

---

## ✅ Requisitos funcionales

| Categoría | Funcionalidad | Descripción | Prioridad |
|------------|----------------|--------------|------------|
| Autenticación | Login y logout | Basado en JWT, sesión persistente | Must-have |
| Gestión de productos | CRUD de productos y stock | Con conexión al WMS Service | Must-have |
| Órdenes y facturación | Listado y gestión de pedidos | Conexión al Billing Service | Must-have |
| Usuarios | CRUD de usuarios del tenant | Roles: `admin`, `staff`, `viewer` | Must-have |
| Notificaciones | Configurar emails / webhooks | Integra con Notification Service | Should-have |
| Pagos | Configurar pasarelas (Stripe/MP) | Integra con Payment Adapter | Should-have |
| Dashboard | Resumen de métricas | Ventas, stock, clientes | Could-have |
| Configuración | Ajustes del tenant | Tema visual, logo, plan | Could-have |

---

## 🧩 Arquitectura del proyecto

src/  
├── app/  
│ ├── core/ # Servicios globales, interceptors, guards  
│ ├── shared/ # Componentes y pipes reutilizables  
│ ├── modules/  
│ │ ├── dashboard/  
│ │ ├── products/  
│ │ ├── orders/  
│ │ ├── users/  
│ │ ├── notifications/  
│ │ ├── payments/  
│ │ └── settings/  
│ ├── layouts/ # Plantillas visuales (sidebar, topbar)  
│ ├── app-routing.module.ts  
│ ├── app.component.ts  
│ └── app.module.ts  
├── assets/  
│ ├── logos/  
│ ├── themes/  
│ └── icons/  
├── environments/  
│ ├── environment.ts  
│ └── environment.prod.ts


---

## ⚙️ Dependencias principales

| Paquete                           | Uso                                                          |
| --------------------------------- | ------------------------------------------------------------ |
| `@angular/core`                   | Framework base                                               |
| `@angular/material`               | Componentes UI base                                          |
| `primeng`                         | Componentes visuales empresariales (DataTable, Dialog, etc.) |
| `primeicons`                      | Iconos integrados                                            |
| `ngx-toastr`                      | Notificaciones visuales                                      |
| `ngx-permissions`                 | Control de roles y permisos                                  |
| `@ngxs/store`                     | State management                                             |
| `ngx-feature-flag` (o custom SDK) | Integración con Unleash                                      |
| `jwt-decode`                      | Decodificación de tokens                                     |
| `rxjs`                            | Programación reactiva                                        |

---

## 🧠 Patrones y buenas prácticas

- **Smart/Dumb Components:**  
  Mantener componentes "tontos" (solo presentación) y contenedores (manejan datos).
- **Lazy Loading por módulo:**  
  Cada feature se carga de forma diferida.
- **Guards y resolvers:**  
  Control de acceso y precarga de datos.
- **Interceptors:**  
  Manejo de JWT, loaders y errores globales.
- **Services centralizados:**  
  Cada módulo tiene su propio servicio (`ProductsService`, `OrdersService`, etc.).

---

## 🔄 Flujos principales

### 1. **Autenticación**
- Formulario de login (`/auth/login`).
- Llama al endpoint `POST /auth/login`.
- Guarda el JWT y el `tenant_id`.
- Inicia guardias (`AuthGuard`) y redirección a `/dashboard`.

### 2. **Gestión de productos**
- Lista con `p-table` (PrimeNG DataTable) + filtros.
- CRUD con `Dialog` + `ReactiveFormsModule`.
- Conexión a `/wms/products`.

### 3. **Gestión de pedidos**
- Listado con estado (chips color).
- Acciones condicionales (cancelar, emitir factura).
- Integración con `/billing/orders` y `/billing/invoices`.

### 4. **Usuarios**
- Tabla con `p-table`, CRUD con roles.
- Permisos visibles según `role` (`ngx-permissions`).

### 5. **Configuración**
- Panel de administración de tenant.
- Configura branding, idioma, moneda y flags activos.

---

## 🎨 UI / UX Guidelines

- **Color palette:**  
  Basada en Material + tonalidades neutras (grises, azules).
- **Spacing:**  
  Usar `p-m-2` y `p-p-2` de PrimeFlex.
- **Tipografía:**  
  `Inter`, `Roboto`, o similar.
- **Componentes clave:**  
  - `p-table` para listados.
  - `p-dialog` para formularios modales.
  - `p-toast` o `ngx-toastr` para feedback.
  - `p-toolbar` y `p-menubar` para layout.
- **Dark mode:**  
  Soporte opcional vía `theme-dark.scss`.

---

## 🧪 Feature Flags (Admin)

| Flag | Descripción | Alcance |
|------|--------------|---------|
| `admin.billing_enabled` | Activa módulo de facturación | Tenant |
| `admin.notifications_enabled` | Habilita sección de notificaciones | Tenant |
| `admin.payments_configurable` | Permite editar pasarelas | Tenant |
| `admin.multi_location_stock` | Muestra vista de almacenes | Tenant |
| `admin.experimental_dashboard` | Activa nuevo dashboard | Global |
| `admin.theme_customization` | Cambiar tema visual por tenant | Tenant |

---

## 🧭 Recomendaciones finales

- Aplicar **OnPush Change Detection** donde sea posible.
- Usar **standalone components** para reducir complejidad de módulos.
- Mantener **naming consistente** (`AdminProductListComponent`, etc.).
- Desarrollar en **modo Feature-driven Development (FDD)**.
- Preparar configuración de **CI/CD** (lint, test, build, deploy).
