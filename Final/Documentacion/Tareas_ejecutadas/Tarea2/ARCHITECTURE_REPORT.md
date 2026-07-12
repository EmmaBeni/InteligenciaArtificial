# Informe de Arquitectura de la aplicación pickndrive-ia

Este documento sintetiza la arquitectura observada del repositorio pickndrive-ia con base en los hallazgos de los módulos de explorer y researcher. No se modifican archivos del repositorio; este informe sirve como guía de alto nivel para entender, mantener y evolucionar la solución.

## 1. Stack tecnológico (stack)
- Frontend principal: React 19 con TypeScript.
- Build/dev tooling: Vite como servidor y bundler.
- Estilo: Tailwind CSS v4.
- Enrutamiento y datos:
  - TanStack Router (enrutamiento tipado y generativo).
  - TanStack Query para manejo de datos (cache/Fetch).
  - Axios para llamadas a API.
- UI: Radix UI (componentes como avatar, diálogos, pestañas, etc.); envoltorios de UI estilo similar a ShadCN/UI.
- Utilidades y utilitarios: date-fns, react-day-picker; lucide-react para iconos.
- Gestión de dependencias y monorepo: pnpm; presencia de pnpm-workspace.yaml y scripts en package.json.
- Estructuras relacionadas: archivo ARCHITECTURE_REPORT.md/README con notas de arquitectura; routeTree.gen.ts para el árbol de rutas generadas y router.ts para la configuración del enrutador.

> Observación: Todo el stack está orientado a una SPA modular, tipada y orientada a una ruta tipada con separación clara de responsabilidades (UI, hooks, services, routes).

## 2. Estructura de carpetas y archivos relevantes
- Nivel raíz
  - Archivos y carpetas típicos de un monorepo: .git, .github, package.json, pnpm-workspace.yaml, public, vite.config.ts, etc.
  - README.md y ARCHITECTURE_REPORT.md (documentación de alto nivel).
  - Archivos de configuración de linters/formatters y scripts de desarrollo (posibles eslint, prettier, husky, etc.).
- Carpeta src/ (aplicación frontend)
  - App.tsx y main.tsx: puntos de entrada de la app.
  - index.css y App.css: estilos globales.
  - assets/: recursos estáticos.
  - components/: componentes reutilizables de UI.
  - pages/: componentes de nivel de página (vistas).
  - routes/: definiciones y estructuras de rutas (probablemente contiene archivos para layout y agrupación de rutas).
  - routeTree.gen.ts: árbol de rutas generado (tipado) para TanStack Router.
  - router.ts: configuración del enrutador.
  - services/: capa de acceso a APIs (lógica de llamadas HTTP vía Axios).
  - constants/: valores constantes usados a lo largo de la app.
  - hooks/: hooks personalizados para la app.
  - types/: definiciones de tipos e interfaces TypeScript.
  - lib/: utilidades o bibliotecas específicas de la app.
  - tsconfig*.json y vite-env.d.ts: configuración de TypeScript y entorno de Vite.
- Otros archivos relevantes en raíz
  - Configuración de linting/formatting (eslint, prettier), scripts de desarrollo (dev, build, lint, format, type-check).

## 3. Patrones y convenciones observadas
- Arquitectura modular: separación clara entre componentes, hooks, services y routes.
- Enrutamiento tipado y generativo: uso de routeTree.gen.ts junto con router.ts para una navegación fuertemente tipada (TanStack Router).
- Capa de servicios para API: carpeta services para encapsular llamadas Axios.
- Tipado fuerte: carpeta types para interfaces y tipos; uso de TS en toda la base de código.
- Consistencia de herramientas: uso de ESLint y Prettier; scripts de formato y lint; husky para hooks de commit (según hallazgos del researcher).
- Configuración de workspace: presencia de pnpm-workspace.yaml, lo que sugiere un monorepo que podría alojar múltiples paquetes; este frontend parece ser una de las unidades del workspace.

## 4. Estructura de carpetas detallada (resumen)
- root/
  - .git, .github, package.json, pnpm-workspace.yaml, vite.config.ts, README.md, ARCHITECTURE_REPORT.md, etc.
- src/
  - App.tsx, main.tsx
  - index.css, App.css
  - assets/
  - components/
  - pages/
  - routes/
  - routeTree.gen.ts
  - router.ts
  - services/
  - constants/
  - hooks/
  - types/
  - lib/
  - tsconfig*.json, vite-env.d.ts

## 5. Dependencias (alto nivel, por función)
- Core/Runtime:
  - React 19, TypeScript, Vite.
- UI y componentes:
  - Tailwind CSS v4, Radix UI (y envoltorios tipo ShadCN/UI).
- Enrutamiento y datos:
  - TanStack Router (navegación tipada), TanStack Query (datos/cache).
  - Axios (calls HTTP a APIs).
- Utilidades y helpers:
  - date-fns, react-day-picker, lucide-react.
- Infraestructura y monorepo:
  - pnpm (pnpm-workspace.yaml).
- Calidad y tooling:
  - ESLint, Prettier, Husky (para hooks de commit), scripts de lint/format/type-check.

Notas: Los nombres exactos de librerías y versiones pueden variar ligeramente en el repositorio, pero la configuración observada se alinea con estas dependencias y su propósito.

## 6. Riesgos y mitigaciones
- Complejidad del monorepo con pnpm:
  - Riesgo: coordinación entre paquetes puede ser compleja, posibles problemas de hoisting y resolución de dependencias.
  - Mitigación: mantener scripts y configuración coherentes; usar herramientas de verificación en CI para asegurar consistencia entre paquetes.
- Enrutamiento tipado y generación de routeTree.gen.ts:
  - Riesgo: si la generación de rutas no se sincroniza con cambios en routes/pages, pueden surgir tipos rotos.
  - Mitigación: automatizar regeneración de routeTree en pasos de CI o con hooks de pre-commit; pruebas de flujo de navegación.
- Gestión de estado global:
  - Riesgo: dependencia excesiva en hooks distribuidos puede generar re-renders y dificultad de depuración.
  - Mitigación: considerar una capa de store ligera (Context + Reducer) para estado compartido crítico.
- Caching y consistencia de datos:
  - Riesgo: si no hay un mecanismo centralizado de caché, diferentes hooks pueden hacer fetch duplicados y estados desincronizados.
  - Mitigación: evaluar TanStack Query como capa de cache centralizada o servicio dedicado con caching.
- Pruebas y calidad:
  - Riesgo: ausencia de pruebas claras para hooks y componentes clave.
  - Mitigación: incorporar pruebas unitarias e de integración progresivamente, con cobertura razonable.

## 7. Comandos relevantes (prácticos)
- Instalación de dependencias y setup del workspace:
  - pnpm install
  - pnpm -w install (si necesario en el contexto de un monorepo)
- Desarrollo y construcción:
  - pnpm run dev  (o npm run dev / yarn dev, según el gestor)
  - pnpm run build
- Linter y formateo:
  - pnpm run lint
  - pnpm run format
- Tipado y compilación:
  - pnpm run typecheck
- Pruebas (si se añaden):
  - pnpm run test
- Observabilidad/opinión de entorno:
  - Revisar logs en la consola; configurar Toaster para notificaciones frente a errores de API.

## 8. Patrones usados
- Arquitectura modular: separación entre UI, hooks, services, rutas, tipos y utilidades.
- Enrutamiento tipado y generativo (routeTree.gen.ts + router.ts).
- Separación de concerns: capa de APIs en services, capa de datos/ hooks para lectura de datos, UI en components/pages.
- Tipado fuerte: uso de TypeScript en toda la base; entidades definidas en /types y consumidas en hooks/ servicios.
- Consistencia de herramientas: ESLint/Prettier, scripts de formato y lint, Husky para hooks de commit.
- Monorepo estructurado con pnpm, sugiriendo posibles múltiples paquetes bajo un mismo workspace.

## 9. Recomendaciones de mejora (priorizadas)
Prioridad alta
- 9.1. Introducir un store global ligero (Reducer + Context)
  - Propósito: centralizar el estado compartido (usuario, items list, estado de carga, errores) y evitar prop drilling.
  - Acción: crear src/store/index.tsx con un Reducer y un Provider raíz; envolver App en el RootProvider.
- 9.2. Centralizar la capa de datos
  - Propósito: estandarizar llamadas a APIs, manejo de errores y posibles cachés.
  - Acción: crear src/services/vehicleService.ts, src/services/userService.ts; implementar un cliente API común con manejo de errores y timeouts.
- 9.3. Aprovechar layout y guards de rutas
  - Propósito: evitar duplicación de UI persistente y aplicar reglas de autenticación.
  - Acción: crear un LayoutRoot que envuelva las rutas con Header y Toaster; aplicar guards si aplica.

Prioridad media
- 9.4. Desacoplar LandingPage en containers/presentational
  - Acción: extraer lógica de negocio a hooks/containers y hacer LandingPage más enfocado en renderizado.
- 9.5. Refinar tipos y entidades
  - Acción: consolidar interfaces Vehicle, User y otras en /types y usarlas en hooks y services para consistencia.

Prioridad baja
- 9.6. Pruebas y cobertura
  - Acción: añadir pruebas unitarias para hooks clave y componentes; pruebas de integración para rutas y flujos de usuario.
- 9.7. Accesibilidad e internacionalización (i18n)
  - Acción: revisar accesibilidad (aria, labels) y evaluar plan de i18n para equipos multilingües.

## 10. Siguientes pasos propuestos
- Paso 1: evaluar y acordar una estrategia de estado global (Context+Reducer) y un layour común para la app.
- Paso 2: iniciar implementación de servicios centralizados y refactor de hooks para consumirlos.
- Paso 3: refactor de LandingPage para separar contenedores/presentación y mejorar testabilidad.
- Paso 4: agregar pruebas unitarias básicas y configurar pipelines de CI para lint/test.
- Paso 5: documentar con ejemplos de cómo añadir nuevas rutas y componentes, y mantener la ruta generada routeTree.gen.ts sincronizada.

Notas finales
- Este informe se apoya en los hallazgos observados y propone mejoras accionables para evolucionar la arquitectura hacia una base de código más mantenible, escalable y con mejor capacidad de prueba. Si quieres, puedo detallar un plan de implementación paso a paso con estructuras de archivos y tipos concretos para empezar a implementar el store y los servicios sin tocar la lógica de negocio existente.