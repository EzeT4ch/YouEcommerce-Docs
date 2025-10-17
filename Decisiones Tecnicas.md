
## Requisitos

### Funcionales (Producto)

*Imprescindible*
- Módulo de gestión de productos: CRUD de productos, categorías, variantes.
- Módulo de facturación: generación de comprobantes, cálculo de totales, impuestos.
- Módulo de gestión de pedidos y checkout: carrito, cálculo de costos, estados de pedido.
- Módulo de usuarios: autenticación, roles (admin, cliente), gestión de sesión.
- Arquitectura modular desacoplada por dominios funcionales (microservicios o módulos desacoplables).
- API REST o GraphQL para comunicación entre módulos y exposición externa.
- Mecanismo de integración con pasarelas de pago (Mock inicial + interfaz estándar).
- Base de datos transaccional (ej: SqlServer).

*Debería hacerse*
- Panel administrativo reutilizable (frontend modular o basado en componentes compartidos).
- Módulo WMS básico: inventario, ubicación de stock, ajustes.
- Multi-tenant básico (soporte para múltiples tiendas/clientes en un mismo entorno).
- Módulo de notificaciones: emails y/o webhooks.
- Integración básica con sistemas de envío (API de terceros o módulo dummy).

*Se podria hacer*
- Editor visual de plantillas para tiendas.
- Catálogo público embebible (iframe o widget JS).
- Webhooks/eventos para integraciones externas.

*No lo haremos (en esta etapa inicial)*
- App móvil nativa.
- Machine learning para recomendaciones.
- Automatizaciones complejas o flujos personalizables por cliente.

### No Funcionales

*Imprescindible*
- Modularidad y reutilización explícita en el diseño desde el primer módulo.
- Bajo costo de mantenimiento y facilidad de despliegue.
- Escalabilidad horizontal desde los servicios core (auth, facturación, productos).
- Seguridad mínima: autenticación segura (JWT o similar), control de acceso.
- Código base orientado a reutilización en múltiples proyectos (internos o de clientes).

*Deberiamos hacer*
- Despliegue automatizado (CI/CD básico).
- Logging centralizado.
- Trazabilidad de errores y métricas.

## Method

La arquitectura propuesta se basa en una estrategia de modularización desde el diseño, permitiendo construir múltiples productos a partir de servicios desacoplados.

### Estilo de arquitectura

* Arquitectura orientada a servicios (Service-Oriented Architecture)
* Implementación basada en módulos independientes (microservicios o monolito modular según capacidad inicial)
* Cada módulo implementado bajo principios de Domain-Driven Design (DDD) y Clean Architecture
* Comunicación principal vía HTTP/REST en etapa inicial; eventos asincrónicos con colas se incorporarán progresivamente
* Autenticación centralizada con tokens JWT y autorización basada en roles
* Multi-tenant desde el diseño (nivel base de cada servicio)

### Componentes

- `auth-service`: Registro, login, autorización, gestión de usuarios y clientes
- `billing-service`: Lógica de facturación, emisión de comprobantes, impuestos
- `wms-service`: Inventario, entradas/salidas, disponibilidad de productos
- `notification-service`: Envío de notificaciones por email/webhook
- `payment-gateway-adapter`: Wrapper unificado para integraciones con Stripe, MercadoPago, etc.
- `saas-ecommerce-core`: Aplicación orquestadora del SaaS principal (frontend+backend)

### Diagrama general de arquitectura

![[Pasted image 20251017115233.png]]

## Implementación

### Organización de código

* Monorepo inicial (con posible evolución a multi-repo)
* Un proyecto backend por módulo (servicio), cada uno con:
  - `Application`: casos de uso
  - `Domain`: entidades, agregados, lógica de negocio
  - `Infrastructure`: persistencia, servicios externos
  - `API`: controladores HTTP, validaciones, DTOs
  - `Shared`: Utilidades para entre capas
* Angular como frontend compartido con:
  - App shell
  - Módulos cargables dinámicamente (admin, tienda, configuración)
  - Reutilización de componentes vía biblioteca interna

Ejemplo de estructura inicial del monorepo:
- /saas-platform/  
	- /services/  
		- /auth-service/  
		- /billing-service/  
		- /wms-service/  
		- /notification-service/  
		- /payment-adapter/  
	- /apps/  
		- /saas-orchestrator/ # Backend de tienda principal  
		- /admin-frontend/ # Angular  
		- /storefront-frontend/ # Angular  
	- /shared/  
/config/  
/docker/

### Secuencia de desarrollo (MVP inicial)

1. **Infraestructura base**
   - Docker compose para cada servicio con SQL Server
   - Configuración mínima de networking y variables de entorno

2. **Módulo Auth (servicio independiente)**
   - Registro/Login por email + JWT
   - Gestión básica de usuarios, tenants y roles
   - Feature flag para onboarding flows (optativo)

3. **Módulo Billing**
   - CRUD de órdenes y generación de comprobantes
   - Cálculo de totales e impuestos simulados

4. **Módulo WMS**
   - CRUD de productos y stock
   - Movimiento de inventario básico (entrada/salida)

5. **Módulo Notifications**
   - Template básico + envío por email simulado
   - Webhook en cola para simular comportamiento real

6. **Payment Adapter**
   - Dummy provider con estructura para integrar Stripe/MercadoPago

7. **Orquestador (SaaS Principal)**
   - API unificada para frontend
   - Frontend admin y storefront inicial
   - Configuración multi-tenant mínima
   - Uso de Feature Flags para alternar funciones en pruebas A/B

8. **Integración progresiva**
   - Backend conecta módulos vía HTTP REST
   - Interfaz común de logs y errores
   - Despliegue de entorno de staging

### Feature Flag Engine

- Inicial: `Unleash` (open source, auto hosteado)
- Flags por tenant, entorno o versión de módulo
- Uso: habilitar/deshabilitar funciones en tienda por cliente, o realizar test A/B

### Milestones

El roadmap está diseñado en fases iterativas para permitir entregas incrementales con foco en funcionalidad usable y reutilizable desde el inicio.

### Fase 1 - Infraestructura base (Semana 1 a 3)

- Repositorio inicial con estructura modular
- Desarrollo de librería Shared interna
- Entorno de desarrollo (Docker + SQL Server)
- Proyecto base de Angular + módulos en blanco
- Setup de Feature Flags (Unleash) en modo básico
- Documentación del stack base

---

### Fase 2 - Módulo Auth + Orquestador MVP (Semana 4 a 7)

- Servicio Auth funcional (login, registro, JWT, roles, Identity)
- Orquestador (API gateway / backend SaaS) con endpoints iniciales
- Frontend de login y gestión básica de usuarios
- Multi-tenant básico (con separación por tenant_id)

**Meta**: tener usuarios funcionando y login operativo

---

### Fase 3 - Productos + Inventario (WMS básico) (Semana 8 a 11)

- Servicio WMS con CRUD de productos y stock
- Orquestador con endpoints para frontend de catálogo
- Admin frontend para gestión de productos
- Mostrar catálogo público simple

**Meta**: poder cargar productos y mostrarlos

---

### Fase 4 - Checkout + Facturación (Semana 12 a 16)

- Servicio Billing: órdenes, cálculo de totales, impuestos mockeados
- Frontend de carrito de compras
- Checkout y generación de "factura"
- Uso de feature flag para activar checkout por tienda

**Meta**: flujo básico de compra

---

### Fase 5 - Notificaciones + Integración de pagos (Semana 17 a 20)

- Servicio de notificaciones por email (simulado / consola)
- Adapter de pagos dummy (estructura lista para MercadoPago/Stripe)
- Simulación de cobro y confirmación de pedido

**Meta**: cerrar flujo de compra con cobro simulado y notificación

---

### Fase 6 - Deploy inicial y testeo real (Semana 21+)

- Entorno de staging desplegado
- Documentación técnica completa
- Pruebas manuales con usuarios reales
- Feedback loop (qué funciona, qué no, qué se reutiliza)

---


