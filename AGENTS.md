# Guía del agente — Ferretería Platform

## Alcance del repositorio

- Este repositorio raíz coordina dos **repositorios Git independientes**: `api-ferreteria/` (NestJS 12) y `front-ferretria/` (Angular 22). Son submódulos, no paquetes de un workspace npm.
- Los cambios de aplicación pertenecen al repositorio del submódulo correspondiente. Los cambios de especificación, documentación global y referencias de submódulos pertenecen a la raíz.
- No hay actualmente una carpeta `specs/`. Cuando una tarea necesite una especificación funcional, usa la convención `specs/features/<nombre>.md` en la raíz y crea las carpetas necesarias.
- La integración entre el frontend y la API debe tratarse como una tarea explícita: no asumir que ya hay endpoints consumidos o una configuración HTTP compartida.

## Antes de trabajar

1. Revisa `git status --short` en la raíz y también dentro de cada submódulo que vayas a tocar (`git -C api-ferreteria status --short`, `git -C front-ferretria status --short`). La raíz solo registra el commit fijado por cada submódulo; revisa el estado anidado antes de editar o cambiar ramas.
2. Conserva los cambios staged, unstaged y sin seguimiento que ya existan. No los restaures, descartes, sobrescribas ni incluyas en un commit ajeno a la tarea.
3. Lee y sigue las instrucciones locales aplicables: `api-ferreteria/AGENTS.md` para el backend y `front-ferretria/AGENTS.md` para el frontend. Esas guías describen los patrones y comandos propios de cada aplicación.
4. No ejecutes instalación, build o pruebas desde la raíz: entra al submódulo correcto. No agregues scripts o dependencias de workspace en la raíz sin que el diseño lo requiera.

## Ramas y flujo de una feature

Para una funcionalidad transversal, coordina la misma rama lógica —por ejemplo, `feature/ventas`— en la raíz y en cada submódulo afectado. Cada repositorio tiene su propio historial, rama, commit, remoto y publicación.

1. Si corresponde, documenta primero el alcance en `specs/features/<feature>.md` desde la rama de feature de la raíz.
2. Implementa y verifica el backend dentro de `api-ferreteria/` cuando la feature lo requiera.
3. Implementa y verifica el frontend dentro de `front-ferretria/` cuando la feature lo requiera.
4. Publica los commits de cada submódulo antes de integrar sus nuevos hashes en la raíz.
5. En la raíz, revisa `git status` y registra explícitamente solo los paths relacionados (`api-ferreteria`, `front-ferretria` y/o `specs`). El commit raíz actualiza las referencias exactas de submódulo; no contiene los archivos internos de los proyectos.

Antes de cambiar ramas, confirma que el árbol de trabajo correspondiente está limpio o identifica claramente los cambios existentes. Tras un clon recursivo, un submódulo puede estar en `HEAD` separado. Si `main` existe localmente, actualízala con `git pull --ff-only` antes de crear la rama de feature; si no existe, créala desde `origin/main` (`git switch -c main --track origin/main`). Si la rama de feature ya existe, cámbiate a ella en vez de intentar crearla otra vez. Nunca muevas una rama para ocultar cambios locales.

No ejecutes `git submodule update --remote` ni cambies el commit fijado de un submódulo salvo que la tarea sea actualizarlo deliberadamente. No hagas commit, push, merge ni cambies ramas automáticamente si el usuario no lo pidió; deja los cambios en el repositorio correcto y resume qué commits/punteros serían necesarios.

## Pruebas y convenciones

- Antes de terminar, corre las verificaciones apropiadas dentro de cada proyecto modificado y reporta los comandos y resultados.
- Frontend: `cd front-ferretria && npm run build && npm test`. No asumas que existe un script `lint` en el frontend.
- Backend: `cd api-ferreteria && npm run build`; según el cambio, usa también `npm run lint`, `npm run test` y/o `npm run test:e2e`. Las pruebas e2e requieren PostgreSQL y configuración `.env`.
- No muestres ni registres secretos de `.env`; usa `.env.example` como referencia de configuración.
- Mantén las migraciones de base de datos en el backend y sigue su guía local. No habilites sincronización automática del esquema como sustituto de migraciones.
- Al informar resultados, especifica claramente si el cambio corresponde a la raíz, al backend, al frontend o a más de uno.
