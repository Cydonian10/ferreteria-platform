# Ferretería Platform

Plataforma de gestión para ferretería. Este repositorio coordina dos proyectos independientes conectados como **Git submodules**; no es un monorepo npm ni comparte una instalación de dependencias.

| Ruta | Proyecto | Tecnología |
| --- | --- | --- |
| `api-ferreteria/` | API y lógica de negocio | NestJS 12, TypeORM y PostgreSQL |
| `front-ferretria/` | Aplicación web | Angular 22, TypeScript y Tailwind CSS 4 |
| `specs/` | Especificaciones de funcionalidades (convención para crear cuando se necesite) | Markdown |

Cada submódulo es un repositorio Git independiente. La raíz mantiene documentación/especificaciones y registra el commit exacto de cada submódulo.

## Requisitos

- Git con acceso a los repositorios configurados en `.gitmodules`.
- Node.js `22.20` o superior y npm. El frontend declara npm `11.16.0` como su gestor de paquetes.
- Docker Compose para iniciar PostgreSQL durante el desarrollo del backend.

## Clonar el proyecto

Clona la raíz junto con sus submódulos:

```bash
git clone --recurse-submodules git@github.com:Cydonian10/ferreteria-platform.git
cd ferreteria-platform
```

Si ya clonaste la raíz sin submódulos, inicialízalos con:

```bash
git submodule update --init --recursive
```

## Ejecutar el backend

Desde la raíz del repositorio:

```bash
cd api-ferreteria
npm install
cp .env.example .env
docker compose up -d
```

Revisa los valores de `.env` y asegúrate de que PostgreSQL esté listo. Para una base nueva, ejecuta las migraciones y luego inicia NestJS:

```bash
npm run migration:run
npm run start:dev
```

La API queda disponible en `http://localhost:3000` y Swagger en `http://localhost:3000/docs`. Los detalles de configuración, arquitectura y migraciones están en [`api-ferreteria/README.md`](api-ferreteria/README.md).

## Ejecutar el frontend

En otra terminal, desde la raíz:

```bash
cd front-ferretria
npm install
npm start
```

Angular sirve la aplicación en `http://localhost:4200`. El frontend y backend se instalan, ejecutan y prueban por separado. La integración HTTP entre ambos debe implementarse/configurarse como parte de las funcionalidades que la requieran.

## Desarrollo y verificaciones

Ejecuta las verificaciones desde la raíz con estos comandos (cada uno entra al submódulo correcto):

```bash
# Frontend (front-ferretria)
(cd front-ferretria && npm run build)
(cd front-ferretria && npm test)

# Backend (api-ferreteria)
(cd api-ferreteria && npm run build)
(cd api-ferreteria && npm run lint)
(cd api-ferreteria && npm run test)
(cd api-ferreteria && npm run test:e2e)
```

El backend requiere PostgreSQL y las variables de entorno para ejecutar las pruebas e2e. Consulta los README de cada submódulo para comandos adicionales y detalles de sus pruebas.

## Flujo de trabajo con submódulos

Una funcionalidad que afecte varios proyectos se desarrolla en repositorios separados. Por ejemplo, para una feature `ventas`:

1. **Raíz:** crea/usa la rama `feature/ventas`. Si la feature necesita especificación, guárdala en `specs/features/ventas.md` (la carpeta `specs/` todavía no existe; créala al añadir la primera especificación).
2. **Backend:** crea/usa `feature/ventas` dentro de `api-ferreteria`, implementa la API, prueba y publica sus commits.
3. **Frontend:** crea/usa `feature/ventas` dentro de `front-ferretria`, implementa la UI, prueba y publica sus commits.
4. **Raíz:** registra las nuevas referencias de submódulo y los cambios de especificación en un commit de integración.

Las ramas de los tres repositorios son independientes. Antes de cambiar de rama, revisa el estado en cada repositorio y conserva cualquier cambio local. Los ejemplos parten de `main`; si `feature/ventas` ya existe, usa `git switch feature/ventas` en vez de crearla otra vez. Si un submódulo está en `HEAD` separado y no tiene una rama `main` local, créala desde `origin/main` con `git switch -c main --track origin/main` y actualízala antes de crear la feature.

Desde la raíz del proyecto, crea primero la rama de integración y, opcionalmente, la especificación:

```bash
git status --short
git switch main
git pull --ff-only
git switch -c feature/ventas
```

Si la feature requiere especificación, crea `specs/features/ventas.md` bajo la raíz (por ejemplo, `mkdir -p specs/features`).

Después trabaja y publica cada submódulo por separado. Ejemplo para el backend:

```bash
cd api-ferreteria
git status --short
git switch main
git pull --ff-only
git switch -c feature/ventas
# implementar y verificar
git add <archivos-del-cambio>
git commit -m "feat: implement sales"
git push -u origin feature/ventas
```

Repite el proceso en `front-ferretria`, con un mensaje apropiado, por ejemplo `feat: implement sales UI`. Una vez publicados los cambios de ambos proyectos, vuelve a la raíz y registra las referencias actualizadas:

```bash
cd ..
git status --short
git add api-ferreteria front-ferretria
git add specs  # solo si agregaste o modificaste especificaciones
git commit -m "feat: integrate sales feature"
git push -u origin feature/ventas
```

El commit de la raíz guarda **hashes concretos**, no nombres de ramas. Publica primero los commits de los submódulos para que sus referencias estén disponibles en el remoto antes de publicar la integración. No uses `git submodule update --remote` como rutina: puede mover las referencias a commits distintos de los que probaste.

Para una tarea de un solo proyecto, abre OpenCode en el directorio correspondiente: `opencode` desde la raíz para documentación/especificaciones e integración, `cd api-ferreteria && opencode` para trabajar solo en backend, o `cd front-ferretria && opencode` para trabajar solo en frontend.

En la raíz, el agente puede revisar ambos repositorios y sus `AGENTS.md`; dentro de un submódulo trabaja directamente con el código y las instrucciones de ese proyecto.
