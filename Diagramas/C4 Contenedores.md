````PUML
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

Person(customer, "Cliente", "Compra productos en tiendas SaaS")

System_Boundary(youecommerce, "YouEcommerce Platform") {
    Container(web_app, "Storefront (Angular)", "Frontend", "Interfaz pública para clientes")
    Container(admin_app, "Admin Panel", "Frontend", "Gestión de productos, stock y ventas")
    Container(api_gateway, "API Gateway / Orchestrator", "ASP.NET Core", "Entrada única a los microservicios")
    Container(auth_service, "Auth Service", "ASP.NET Core", "Gestión de usuarios y tenants")
    Container(billing_service, "Billing Service", "ASP.NET Core", "Facturación y órdenes")
    Container(wms_service, "Inventory Service", "ASP.NET Core", "Gestión de stock y productos")
    Container(notification_service, "Notification Service", "ASP.NET Core", "Envío de emails y webhooks")
    Container(payment_service, "Payment Adapter", "ASP.NET Core", "Integración con pasarelas de pago")
    ContainerDb(sql_db, "SQL Server", "Base de datos relacional")
    Container(queue, "Message Queue", "RabbitMQ / Azure Bus", "Eventos asincrónicos (futuro)")
}

Rel(customer, web_app, "Compra productos", "HTTPS")
Rel(customer, admin_app, "Gestiona su tienda", "HTTPS")
Rel(web_app, api_gateway, "Invoca API REST", "JSON/HTTPS")
Rel(admin_app, api_gateway, "Administra datos", "JSON/HTTPS")
Rel(api_gateway, auth_service, "Autenticación", "REST")
Rel(api_gateway, billing_service, "Órdenes y pagos", "REST")
Rel(api_gateway, wms_service, "Stock", "REST")
Rel(api_gateway, notification_service, "Notificaciones", "REST/Eventos")
Rel(billing_service, payment_service, "Procesa pagos", "Webhook/REST")
Rel(billing_service, notification_service, "Envía recibos", "Evento")
Rel(payment_service, billing_service, "Confirma pago", "Webhook")
Rel(api_gateway, sql_db, "Persistencia", "SQL")
@enduml
```
