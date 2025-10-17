# 📦 Auth Service – Detalle Funcional

## 🧭 Rol general
Servicio encargado de la identidad, autenticación y autorización. Es el proveedor de usuarios y tenants para el resto del ecosistema.

---

## ✅ Funcionalidades clave

| Funcionalidad               | Descripción                                           | Priorización |
|----------------------------|-------------------------------------------------------|--------------|
| Registro de usuario        | Alta de usuario con email y contraseña                | Must-have    |
| Inicio de sesión (Login)   | JWT + refresh token (si aplica)                       | Must-have    |
| Gestión de sesiones        | Validación de token activo                            | Must-have    |
| Multi-tenant básico        | Asociación de usuarios a tenants (tiendas/clientes)   | Must-have    |
| Recuperación de contraseña | Token vía email (mock en etapa inicial)               | Should-have  |
| Roles y permisos           | `admin`, `client`, etc.                               | Should-have  |
| Gestión de usuarios        | CRUD de usuarios por tenant                           | Should-have  |
| Auditoría / Logs           | Registro de actividad                                 | Could-have   |
| OAuth2 / Social login      | Google, etc.                                          | Won’t-have   |

---

## 🧩 Entidades principales

### `User`
- `id`: GUID
- `email`: string (único)
- `password_hash`: string
- `full_name`: string
- `tenant_id`: FK a Tenant
- `role`: enum (`admin`, `client`, etc.)
- `is_active`: bool
- `created_at`: datetime

### `Tenant`
- `id`: GUID
- `name`: string
- `plan`: enum (`free`, `pro`)
- `created_at`: datetime
- `status`: enum (`active`, `suspended`)

### `RefreshToken` *(opcional)*
- `id`: GUID
- `user_id`: FK a User
- `token`: string
- `expires_at`: datetime
- `revoked_at`: datetime (nullable)

---

## 🔐 Flujos principales

### Registro
- `POST /auth/register`
- Crea usuario y tenant si no existe
- Retorna JWT

### Login
- `POST /auth/login`
- Verifica credenciales y devuelve JWT

### Validación de sesión
- Middleware verifica el JWT (firma y expiración)

### Roles y permisos
- Claims en el JWT: `tenant_id`, `user_id`, `role`
- Se usa en frontend y backend para aplicar control de acceso

### Multi-tenant seguro
- `tenant_id` se infiere desde el token
- No se acepta `tenant_id` como input del cliente

---

## 🌐 Endpoints

| Método | Ruta                     | Descripción                                        |
| ------ | ------------------------ | -------------------------------------------------- |
| POST   | `/auth/register`         | Registro de usuario y tenant                       |
| POST   | `/auth/login`            | Inicio de sesión                                   |
| GET    | `/auth/me`               | Información del usuario autenticado                |
| POST   | `/auth/logout`           | Invalida el refresh token (si se usa)              |
| POST   | `/auth/forgot-password`  | Inicio de recuperación de contraseña               |
| POST   | `/auth/reset-password`   | Finalización de recuperación                       |
| GET    | `/auth/users`            | Listar usuarios del tenant actual                  |
| POST   | `/auth/users`            | Crear usuario adicional (multiusuario)             |
| DELETE | `/auth/users/{id}`       | Eliminar usuario del tenant                        |
| POST   | `/auth/validate`         | Validar token JWT y extraer claims                 |
| GET    | `/health`                | Health check                                       |
| GET    | `/.well-known/jwks.json` | **(Futuro)** Claves públicas para validación RS256 |
**Notas**  
- Todos los endpoints (salvo `register`, `login`, `forgot/reset`, `health`, `jwks`) requieren `Authorization: Bearer <JWT>`.

---

###  Feature Flags

| Flag                         | Propósito | Alcance |
|-----------------------------|-----------|---------|
| `auth.refresh_tokens`       | Habilita uso de refresh tokens | Global/Tenant |
| `auth.password_reset`       | Activa flujo de recuperación de contraseña | Tenant |
| `auth.roles_advanced`       | Permite roles adicionales y permisos finos | Global |
| `auth.oauth_google`         | Activa login social con Google | Global/Tenant |
| `auth.rate_limit_login`     | Limita intentos de login (anti-abuso) | Global |
| `auth.jwks_enabled`         | Expone JWKS y firma RS256 | Global |

---

## 🛠️ Validación de Tokens – Estrategia Mixta

Para facilitar el uso del servicio por parte de clientes internos y externos, se implementará una validación mixta:

### 1. **Endpoint REST**
- `POST /auth/validate`
- Verifica la firma y expiración del token
- Devuelve claims útiles (`user_id`, `tenant_id`, `role`, `exp`)
- Útil para integraciones sin SDK o clientes externos

### 2. **SDK oficial**
- Librería compartida (`AuthLib.Shared`) para proyectos en .NET
- Permite validación offline con clave secreta compartida (HS256) o clave pública (RS256)
- Extrae y valida claims en el backend

### Futuro: JWKS
- Publicación de clave pública vía `/.well-known/jwks.json` para validación sin contacto

---

## 🛠️ Tecnología sugerida

- `.NET 9` con ASP.NET Core
- `SQL Server` (tablas `Users`, `Tenants`)
- `JWT` (HS256 inicialmente; RS256 a futuro)
- `FluentValidation` para inputs
- `Docker` + health checks básicos
- `Unleash` para flags como login social o gestión de usuarios

---

## 🧪 Consideraciones

- Diseñar con soporte multi-tenant desde el modelo
- Agregar tests unitarios para login, registro y validación de tokens
- Logs de eventos de sesión (login, logout, errores)
- Uso de claims JWT para control de acceso sin llamadas adicionales

