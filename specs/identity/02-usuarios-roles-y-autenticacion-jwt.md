# SPEC 02 — Usuarios, roles y autenticación JWT

> **Estado:** Borrador
> **Depende de:** SPEC 01
> **Repositorios afectados:** `api-ferreteria`
> **Fecha:** 2026-09-23
> **Objetivo:** Incorporar cuentas vinculadas a personas, autenticación JWT de acceso, varios roles globales por usuario y autorización inicial para identidad, productos y ventas.

## Alcance

**Incluye:**

- Evolucionar el usuario actual al modelo de credenciales separado de `Person`, con relación uno a uno obligatoria.
- Login y obtención del usuario actual mediante access token JWT firmado y con expiración configurable.
- Varios roles globales asignables a un usuario; cargar inicialmente `admin`, `cashier`, `seller` y `warehouse`.
- Alta, consulta, modificación, asignación de roles y desactivación de usuarios por un administrador.
- Guards y decoradores NestJS para validar JWT y roles.
- Un comando de bootstrap de un solo uso para crear la empresa inicial, la persona y el primer usuario `admin`.
- Proteger las rutas existentes de usuarios, productos y ventas; vincular la venta al usuario autenticado.
- Una migración TypeORM manual para una base de desarrollo recreada que incluya las tablas de identidad y las tablas `products`, `sales` y `sale_details` que aún necesita la aplicación actual; no se migran los usuarios ni las ventas locales existentes.

**Fuera de alcance (para specs futuras):**

- Refresh tokens, recuperación de contraseña, invitaciones por email, MFA y registro público.
- CRUD de roles personalizados o permisos configurables por módulo.
- Permisos de `cashier` y `warehouse` sobre operaciones de caja e inventario, hasta que esos módulos tengan endpoints.
- Migrar o conservar los datos de la base local actual.

## Modelo de datos

```ts
interface User {
  id: string; // UUID
  personId: string; // FK única a Person
  email: string; // único sin distinguir mayúsculas/minúsculas
  passwordHash: string;
  isActive: boolean;
  lastLoginAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

interface Role {
  code: 'admin' | 'cashier' | 'seller' | 'warehouse';
  name: string;
  description: string | null;
}

interface UserRole {
  userId: string;
  roleCode: Role['code'];
}

interface AccessTokenClaims {
  sub: string; // UUID del usuario
  roles: Role['code'][];
  iat: number;
  exp: number;
}
```

Las credenciales viven solo en `users`; `people` mantiene los datos personales y pertenece obligatoriamente a una empresa. Una persona puede no tener usuario. Los roles son globales y no contienen `company_id`. El catálogo inicial usa los cuatro códigos indicados y admite varias asignaciones por usuario.

La tabla actual `sales` conserva su relación con `users`, pero su FK usa el UUID nuevo. No se cambia el resto del modelo de ventas ni el de productos en esta especificación; la migración los crea con las columnas que ya exponen las entidades actuales.

`POST /api/auth/login` recibe `{ email, password }` y responde con un access token y el perfil autenticado. `GET /api/auth/me` devuelve el usuario y sus roles. No hay registro público ni refresh token. El frontend mantendrá el access token en memoria y lo enviará con `Authorization: Bearer <token>`; al recargar la aplicación se vuelve a iniciar sesión.

`POST /api/users` recibe `personId`, `email`, `password` y una lista `roleCodes`. El administrador puede consultar y editar usuarios, modificar sus roles y desactivarlos. Las contraseñas nunca se almacenan ni retornan en claro. El comando `npm run identity:bootstrap` crea los cuatro roles si faltan y, usando valores de configuración validados, crea empresa, persona y usuario administrador dentro de una transacción. No acepta credenciales como argumentos de línea de comandos y no cambia una instalación que ya tiene administrador.

### Política inicial de acceso

| Recurso/operación | `admin` | `seller` | `cashier` / `warehouse` |
| --- | --- | --- | --- |
| Empresas, tiendas, almacenes, ubicaciones, personas, usuarios y roles | Gestionar | Sin acceso | Sin acceso |
| Productos: `GET` | Sí | Sí | Sin acceso |
| Productos: `POST`, `PATCH` | Sí | No | Sin acceso |
| Ventas: `GET`, `POST` | Sí | Sí | Sin acceso |

Las capacidades de varios roles se combinan. Los roles `cashier` y `warehouse` se crean y asignan, pero no autorizan todavía rutas de operaciones no especificadas. Toda llamada a rutas protegidas sin token válido responde `401`; una identidad autenticada sin el rol necesario responde `403`.

## Plan de implementación

1. Actualizar `env.schema.ts`, `.env.example` y `README.md` para validar y documentar `JWT_ACCESS_SECRET`, `JWT_ACCESS_TTL` y el bootstrap; añadir el servicio de hash de contraseñas y su dependencia Argon2id. Validar configuración inválida al arrancar y que nunca se exponga el hash.
2. Añadir `User`, `Role` y `UserRole` al módulo `identity`, carga idempotente de los cuatro roles y una cadena de migraciones manual que deje las tablas de identidad y las tablas actuales de productos/ventas en una base vacía. Mantener la FK de ventas al usuario. Preparar el reinicio de desarrollo de forma documentada y explícita, sin ejecutarlo automáticamente.
3. Implementar `AuthModule`, login, validación de token y endpoint `/api/auth/me`; probar credenciales válidas, inválidas, expiración y cuenta desactivada.
4. Implementar administración de usuarios y roles mediante CQRS, con rutas bajo `/api/users` y `/api/roles`; restringirlas al administrador y hacer el bootstrap transaccional e idempotente.
5. Aplicar guards a las rutas existentes de productos y ventas según la matriz de acceso. Cambiar la creación de ventas para obtener el usuario desde el principal autenticado y dejar de confiar en `userId` enviado por el cliente.
6. Registrar módulos y documentar login, esquemas de seguridad y respuestas 401/403 en Swagger; dejar el backend arrancando desde la base vacía mediante las migraciones y el bootstrap documentados.

## Criterios de aceptación

- [ ] Un usuario siempre referencia una persona y una persona pertenece a una empresa; `person_id` es único en `users`.
- [ ] El bootstrap crea empresa, persona, usuario administrador y los cuatro roles de forma atómica, y se niega a reemplazar un administrador existente.
- [ ] Un usuario puede tener más de un rol global; el catálogo inicial contiene `admin`, `cashier`, `seller` y `warehouse`.
- [ ] El login válido entrega un JWT firmado con `sub`, `roles`, `iat` y `exp`; el login inválido responde con error genérico sin revelar si existe el email.
- [ ] Contraseñas almacenadas usan Argon2id; los DTOs, logs y respuestas no exponen contraseñas ni hashes.
- [ ] Rutas protegidas responden `401` sin credenciales y `403` cuando los roles no autorizan la operación.
- [ ] La matriz de acceso anterior se cumple en `/api/companies`, `/api/stores`, `/api/warehouses`, `/api/warehouse-locations`, `/api/people`, `/api/users`, `/api/roles`, `/api/products` y `/api/sales`.
- [ ] Crear una venta asocia el usuario del JWT incluso si el cliente envía otro `userId`; el actor no se elige desde el body.
- [ ] Desactivar una cuenta bloquea nuevos logins y solicitudes autenticadas posteriores a la verificación del estado de la cuenta.
- [ ] `npm run identity:bootstrap` valida configuración requerida, no registra secretos y no modifica datos si ya existe un administrador.
- [ ] `npm run build`, `npm run lint` y las pruebas unitarias de guards, login, bootstrap y autorización pasan en `api-ferreteria`.
- [ ] La cadena de migraciones manuales crea las tablas requeridas por las entidades actuales de identidad, productos y ventas en una base de desarrollo vacía, sin `synchronize: true` ni migración de usuarios o ventas heredados.

## Decisiones

- **Sí:** conservar la relación conceptual de ventas con `User`, pero cambiar la cuenta a UUID y derivar al actor del JWT; la base local se recrea y no se migran IDs anteriores.
- **Sí:** usar roles globales y relación muchos-a-muchos `user_roles`, conforme al ERD de referencia.
- **Sí:** cargar cuatro roles conocidos. Solo `admin` y `seller` reciben permisos para las rutas actuales; `cashier` y `warehouse` quedan listos para sus módulos futuros.
- **Sí:** usar Argon2id para almacenar contraseñas y un access token en memoria en el navegador. El alcance no incluye renovación de sesión; recargar requiere login.
- **Sí:** configurar secreto y expiración JWT desde variables validadas por Zod; no fijar secretos en código ni en el repositorio.
- **No:** conservar el antiguo `users` con nombre, email y teléfono como modelo de login. Los datos de desarrollo se reemplazan por el nuevo modelo y el perfil se obtiene de `Person`.
- **No:** permitir registro público, refresh token o administración de roles personalizados.

## Riesgos

| Riesgo | Mitigación |
| --- | --- |
| El reinicio local destruye usuarios y ventas de desarrollo. | Aplicar el reinicio solo después de completar y revisar la migración base; documentar el comando destructivo y nunca incluirlo en comandos automáticos. |
| Un JWT ya emitido conserva roles hasta expirar. | Usar expiración configurable y verificar el estado activo de la cuenta en solicitudes protegidas; tras cambiar roles, la UI inicia una sesión nueva para obtener claims actualizados. |
| El usuario actual de ventas tiene ID numérico y se recibe en el body. | Recrear la base y adaptar la relación a UUID; el handler toma el actor del principal validado, no del DTO. |

## Qué **no** incluye esta especificación

- Refresh tokens, MFA, restablecimiento de contraseñas, invitaciones o registro público.
- Roles y permisos personalizados por empresa.
- Permisos operativos para caja y almacén.
- Migración de usuarios, ventas u otros datos existentes de desarrollo.
