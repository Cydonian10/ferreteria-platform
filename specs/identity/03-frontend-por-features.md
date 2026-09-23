# SPEC 03 — Frontend organizado por features

> **Estado:** Borrador
> **Depende de:** SPEC 01, SPEC 02
> **Repositorios afectados:** `front-ferretria`
> **Fecha:** 2026-09-23
> **Objetivo:** Organizar todas las pantallas Angular por dominio y conectar autenticación, identidad, productos y ventas con esquemas Zod y recursos HTTP reactivos.

## Alcance

**Incluye:**

- Estructura Angular por feature con `data-access/`, `state/`, `pages/` y `components/`.
- Features `auth`, `identity`, `dashboard`, `productos`, `categorias`, `inventario`, `caja`, `ventas`, `proveedores`, `compras`, `reportes` y `configuracion`.
- Pantalla de login, sesión en memoria, guard de rutas e interceptor Bearer para JWT.
- Conexión con los endpoints reales de identidad, productos y ventas.
- Listado de personas con paginación y búsqueda sincronizadas con query params.
- Schemas Zod para validar las respuestas HTTP usadas por el frontend; `httpResource` para lecturas y `HttpClient` para escrituras.
- Conservación de los datos de ejemplo de las secciones que todavía no tienen endpoints.
- Configuración explícita de la URL base de la API y de la integración local entre Angular y NestJS.

**Fuera de alcance (para specs futuras):**

- Crear endpoints backend para categorías, inventario, caja, proveedores, compras o reportes.
- Reemplazar los datos de ejemplo de secciones sin API.
- Persistir el token en `localStorage`, implementar refresh tokens o mantener sesión después de recargar el navegador.
- Rediseñar el shell visual completo o cambiar la identidad visual actual.

## Modelo de datos

Los modelos TypeScript y los schemas Zod reflejan los contratos HTTP confirmados por SPEC 01 y SPEC 02. Ejemplo de lectura tipada y validada:

```ts
const productSchema = z.object({
  id: z.number(),
  name: z.string(),
  price: z.number(),
  description: z.string().nullable().optional(),
  stock: z.number(),
});

readonly products = httpResource(
  () => `${apiBaseUrl}/products`,
  { parse: (data) => z.array(productSchema).parse(data) },
);
```

El ejemplo refleja el contrato actual de `GET /api/products`; si el backend cambia el contrato, el schema y el modelo se actualizan junto con él. El listado de personas valida `Page<Person>` con `{ items, total, page, limit }`. Las consultas `httpResource` permanecen de solo lectura; altas, cambios, login y desactivaciones se realizan con `HttpClient` desde `data-access/`.

Estructura objetivo:

```text
src/app/features/
├── auth/
│   ├── data-access/
│   ├── state/
│   ├── pages/
│   └── components/
├── identity/
│   ├── data-access/
│   ├── state/
│   ├── pages/
│   └── components/
├── productos/
├── categorias/
├── inventario/
├── caja/
├── ventas/
├── proveedores/
├── compras/
├── reportes/
├── configuracion/
└── dashboard/
```

Cada feature mantiene sus propios modelos, schemas, acceso a datos, estado, páginas y componentes del dominio. Las piezas verdaderamente compartidas permanecen en `src/app/shared/` o `src/app/layout/`.

## Plan de implementación

1. Añadir Zod al frontend, configurar `apiBaseUrl` por entorno y definir schemas/modelos de productos, ventas, login, usuario, persona y página de personas. Verificar instalación y compilación sin cambiar rutas.
2. Implementar `AuthApi`, estado/facade de sesión con signals, interceptor Bearer, login y guard de rutas; conectar `POST /api/auth/login` y `GET /api/auth/me`.
3. Implementar el feature `identity` con listados y formularios para personas, usuarios, roles, empresas, tiendas, almacenes y ubicaciones; usar `httpResource` validado en lecturas y `HttpClient` para mutaciones.
4. Implementar paginación y búsqueda de personas desde la UI; propagar cambios de página, tamaño y búsqueda al endpoint y mostrar carga, error, resultados vacíos y total.
5. Mover productos y ventas desde las pantallas genéricas a features propios; integrar `GET /api/products`, sus escrituras disponibles y `GET/POST /api/sales`, validando respuestas con Zod.
6. Mover las secciones restantes a sus features y conservar sus fixtures para categorías, inventario, caja, proveedores, compras, reportes y configuración. Mantener dashboard, shell, navegación y lazy loading.
7. Ajustar rutas, navegación, títulos, guards por rol y estados accesibles; retirar el uso de la pantalla genérica para las rutas migradas y dejar la aplicación compilable y navegable.

## Criterios de aceptación

- [ ] Cada sección enumerada tiene su límite de feature; las rutas ya no dependen de una única `SectionPage` genérica para renderizar dominios distintos.
- [ ] Las features separan acceso a datos, estado, páginas y componentes; los elementos compartidos siguen fuera de ellas.
- [ ] `npm run build` completa correctamente y `npm test` pasa en `front-ferretria`.
- [ ] Zod valida en runtime las respuestas de lecturas reales de autenticación, identidad, productos y ventas; una respuesta inválida se muestra como error de carga y no se trata como dato confiable.
- [ ] Las lecturas GET usan `httpResource` con `parse`; las escrituras usan `HttpClient` y refrescan el estado visible.
- [ ] El interceptor envía el access token como Bearer, la sesión se limpia al cerrar sesión y un reload requiere autenticarse de nuevo.
- [ ] Las rutas protegidas redirigen a login si no hay sesión y las acciones de administración solo se muestran a usuarios con `admin`; el backend sigue siendo la autoridad final del permiso.
- [ ] El listado de personas permite cambiar página, tamaño y búsqueda, y presenta los valores `items`, `total`, `page` y `limit` de la API.
- [ ] El frontend consume la URL base configurada para `/api`; no asume que el servidor Angular y NestJS comparten origen.
- [ ] Las pantallas de categorías, inventario, caja, proveedores, compras, reportes y configuración conservan su contenido de ejemplo hasta que existan endpoints.
- [ ] Formularios, errores, navegación y tablas mantienen accesibilidad por teclado, etiquetas, foco y contraste WCAG AA según `front-ferretria/AGENTS.md`.

## Decisiones

- **Sí:** usar carpetas de dominio en español para las features de negocio, `identity`/`auth` para identidad y acceso, y mantener nombres HTTP en inglés como los endpoints actuales.
- **Sí:** usar `httpResource` únicamente para lecturas y Zod en `parse`; `httpResource` no reemplaza los métodos de escritura.
- **Sí:** mantener fixtures para módulos sin endpoints y conectar solo identidad, productos y ventas en esta entrega.
- **Sí:** mantener el access token en memoria para no guardarlo en almacenamiento persistente; al recargar se requiere un nuevo login mientras no exista refresh token.
- **Sí:** conservar el shell/layout compartido y cargar rutas de features de forma lazy.
- **No:** inventar endpoints o respuestas para completar features aún no implementadas en el backend.
- **No:** mantener `SectionPage` como implementación final de productos, ventas o identidad; la pantalla genérica actual solo ofrece datos de muestra.

## Riesgos

| Riesgo | Mitigación |
| --- | --- |
| La API y Angular corren en orígenes diferentes durante desarrollo. | Configurar `apiBaseUrl` y CORS explícitamente; verificar una llamada real desde el frontend. |
| Los contratos existentes de productos/ventas son más simples que los schemas futuros del documento de diseño PostgreSQL. | Validar el contrato actualmente expuesto y versionar cambios de API junto con el schema Zod, sin anticipar endpoints no implementados. |
| El token en memoria se pierde al refrescar la pestaña. | Redirigir al login con mensaje claro; considerar persistencia o refresh en una especificación posterior. |

## Qué **no** incluye esta especificación

- Nuevos endpoints de backend para caja, compras, inventario, proveedores, categorías o reportes.
- Sustituir datos de muestra en secciones que no tengan API.
- Refresh token, almacenamiento persistente de JWT o recuperación de sesión tras una recarga.
- Una reescritura visual completa del dashboard y del shell.
