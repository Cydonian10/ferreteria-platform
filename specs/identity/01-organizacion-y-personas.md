# SPEC 01 — Organización y personas

> **Estado:** Aprobado
> **Depende de:** Ninguna
> **Repositorios afectados:** `api-ferreteria`
> **Fecha:** 2026-09-23
> **Objetivo:** Incorporar empresas, tiendas, almacenes, ubicaciones y personas vinculadas a una empresa, con mantenimiento lógico y consulta paginada de personas.

## Alcance

**Incluye:**

- El modelo `Empresa → Tienda → Almacén → Ubicación` dentro del módulo NestJS `identity`.
- Personas asociadas obligatoriamente a una empresa.
- Altas, consultas, modificaciones y desactivación lógica de empresas, tiendas, almacenes, ubicaciones y personas.
- Paginación y búsqueda de personas por nombre o documento.
- Migraciones TypeORM manuales y actualización de la documentación del modelo PostgreSQL.
- Rutas HTTP en inglés bajo el prefijo existente `/api`.

**Fuera de alcance (para specs futuras):**

- Usuarios, credenciales, roles, JWT y permisos, que pertenecen a SPEC 02.
- Stock, lotes, movimientos de inventario y operaciones de ventas.
- Borrado físico o migración de registros personales existentes.
- Multiempresa aislada por políticas RLS; esta entrega administra una estructura de empresas, pero no define aislamiento de datos entre ellas.

## Modelo de datos

Las tablas usan UUID, `created_at`, `updated_at` y `deleted_at`. Los registros con `deleted_at` no se incluyen en consultas normales y no se eliminan físicamente.

```ts
interface Company {
  id: string;
  legalName: string;
  tradeName: string | null;
  taxId: string; // único
  address: string | null;
  phone: string | null;
  email: string | null;
  currencyCode: string; // PEN por defecto
  timezone: string; // America/Lima por defecto
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

interface Store {
  id: string;
  companyId: string;
  code: string; // único dentro de la empresa
  name: string;
  address: string | null;
  phone: string | null;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

interface Warehouse {
  id: string;
  storeId: string;
  code: string; // único dentro de la tienda
  name: string;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

interface WarehouseLocation {
  id: string;
  warehouseId: string;
  code: string; // único dentro del almacén
  name: string;
  isSaleable: boolean;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

interface Person {
  id: string;
  companyId: string; // obligatorio
  documentType: string; // obligatorio
  documentNumber: string; // obligatorio
  firstName: string;
  lastName: string | null;
  phone: string | null;
  email: string | null;
  address: string | null;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

interface Page<T> {
  items: T[];
  total: number;
  page: number;
  limit: number;
}
```

`tax_id` es único. Los códigos de tienda, almacén y ubicación son únicos dentro de su padre. La combinación de tipo y número de documento es única entre personas activas. Las claves foráneas impiden crear hijos sin su padre.

Rutas base: `/api/companies`, `/api/stores`, `/api/warehouses`, `/api/warehouse-locations` y `/api/people`. Cada recurso expone `GET` de colección, `GET /:id`, `POST`, `PATCH /:id` y `DELETE /:id`; `DELETE` asigna `deleted_at` y responde `204`, sin borrar físicamente. Las colecciones devuelven solo registros activos.

`GET /api/people` admite `page` (por defecto `1`), `limit` (por defecto `20`, máximo `100`) y `search` opcional. `search` encuentra coincidencias parciales en nombres, apellidos o número de documento. La respuesta sigue `Page<Person>` y tiene orden estable por `id`.

## Plan de implementación

1. Actualizar `api-ferreteria/docs/database/01-decisions.md`, `02-erd.mmd`, `03-schema-postgresql.sql`, `04-indexes-and-rules.md` y `05-typeorm-migration-plan.md` con `company_id` obligatorio en personas, documento obligatorio y las restricciones descritas aquí. Validación: revisar que el DDL y el ERD coincidan.
2. Añadir entidades TypeORM y una migración SQL manual para empresas, tiendas, almacenes, ubicaciones y personas. Mantener `synchronize: false` y comprobar la migración en una base PostgreSQL desechable.
3. Implementar comandos, consultas y handlers CQRS para empresas y tiendas, junto con sus DTO Zod y controladores. Comprobar altas, consultas, cambios y desactivación mediante pruebas de handler y HTTP.
4. Implementar los mismos casos para almacenes y ubicaciones, validando las relaciones con tienda y almacén.
5. Implementar personas y paginación con TypeORM, incluida búsqueda, límites y metadatos; registrar `IdentityModule` en `AppModule` sin romper los módulos actuales.
6. Documentar las rutas y respuestas en Swagger y dejar la API ejecutable con la migración aplicada en PostgreSQL.

## Criterios de aceptación

- [ ] El DDL, el ERD y las entidades describen las cinco estructuras con los mismos campos, relaciones y restricciones.
- [ ] Una persona no se puede crear sin empresa, tipo de documento, número de documento y nombre.
- [ ] No se aceptan identificadores fiscales ni códigos duplicados dentro de sus ámbitos definidos.
- [ ] Las operaciones de lectura y escritura funcionan para los cinco recursos y responden con errores HTTP apropiados ante IDs inexistentes o conflictos.
- [ ] `DELETE` establece `deleted_at`; las lecturas normales dejan de devolver el registro y la fila permanece en PostgreSQL.
- [ ] `GET /api/people?page=2&limit=10&search=ana` devuelve `items`, `total`, `page` y `limit`, y filtra por nombre, apellido o documento.
- [ ] La paginación rechaza páginas o límites no positivos y límites superiores a `100`.
- [ ] `npm run build` y `npm run lint` pasan en `api-ferreteria`, y la migración puede ejecutarse en PostgreSQL limpio.

## Decisiones

- **Sí:** mantener `identity` como límite de dominio para empresa, tienda, almacén, ubicación y persona, tal como pidió el alcance funcional.
- **Sí:** adaptar el esquema de referencia de `api-ferreteria/docs/database/`; en ese diseño la persona no tenía empresa obligatoria y el documento era opcional.
- **Sí:** usar UUID, claves foráneas y `deleted_at`, siguiendo las convenciones del DDL de referencia.
- **Sí:** usar nombres de rutas en inglés para conservar consistencia con `/api/products`, `/api/users` y `/api/sales`; la interfaz puede estar en español.
- **No:** borrar físicamente registros maestros. Los registros con historial se desactivan lógicamente.
- **No:** migrar las personas existentes. El entorno se encuentra en desarrollo y la base local se recreará siguiendo un paso manual documentado.

## Riesgos

| Riesgo                                                                          | Mitigación                                                                                                                                                                                                    |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| El reinicio de la base de desarrollo elimina datos existentes.                  | Documentar el comando destructivo y ejecutarlo manualmente solo contra la base local una vez que la cadena de migraciones necesaria esté completa. No borrar volúmenes desde build, bootstrap ni migraciones. |
| Una persona puede quedar referenciada por un usuario o por operaciones futuras. | Usar desactivación lógica y restringir cambios de empresa/documento cuando existan referencias incompatibles.                                                                                                 |

## Qué **no** incluye esta especificación

- Autenticación, cuentas de usuario, asignación de roles ni JWT; van en SPEC 02.
- Gestión de inventario, compras, caja, stock, lotes o movimientos.
- Aislamiento multiempresa mediante RLS.
- Eliminación física ni conservación/migración de los datos de la base local actual.
