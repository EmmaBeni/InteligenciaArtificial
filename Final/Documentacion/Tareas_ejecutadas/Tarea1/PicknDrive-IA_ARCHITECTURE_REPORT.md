# Architecture Report - Pickndrive IA (React SPA)

Este informe sintetiza la arquitectura observada en el repositorio React ubicado en `/content/workspace/pickndrive-ia`, con foco en stack, estructura de carpetas, patrones usados y recomendaciones de mejora.

## 1. Stack tecnológico
- Frontend: React + TypeScript
- Build y herramientas: Vite, pnpm
- Enrutamiento: TanStack Router (routeTree.gen.ts y router.ts)
- Capa de datos: React Query para consumo y gestión de datos
- Servicios/API: capa src/services para llamadas al backend
- Tipos: src/types para definición de tipos y datatypes
- Observabilidad y UI: Toaster para notificaciones y librerías UI (p. ej. lucide-react para íconos)
- Convenciones de alias: uso de alias @ para imports (configurado en tsconfig/paths)

> Nota: no se detectó manejo explícito de estado global mediante Context o reducers en los fragmentos analizados; la gestión de estado parece centrarse en hooks y data fetching a nivel de página.

## 2. Estructura de carpetas (orientación por módulos)
- src/
  - components/: biblioteca de UI reutilizable
    - subcarpetas mencionadas/ejemplos: rent-my-car, ui, vehicle-details-page, user-tabs
  - pages/: componentes de página que orquestan UI y lógica por ruta
  - routes/: definición de rutas y archivos relacionados (contenido generado para TanStack Router)
  - services/: capa de API para llamadas al backend
  - types/: definiciones de tipos/datatypes usados en toda la app
  - hooks/: lógica de negocio reutilizable y hooks de datos
  - constants/: valores constantes usados globalmente

- Enrutamiento y archivos clave
  - src/router.ts: crea la instancia de TanStack Router usando routeTree
  - src/routeTree.gen.ts: árbol de rutas generado automáticamente por TanStack Router; no debe modificarse manualmente
  - Rutas principales observadas (con dinámicas):
    - /, /about, /auth, /forget-password, /landingPage, /playground, /profile, /publish, /reset-password, /search
    - /vehicle/$documentId y /bookings/$documentId (rutas dinámicas)

## 3. Patrones y prácticas observadas
- Modularidad orientada a características (feature-based): estructura de componentes y páginas organizada por funcionalidad, con rutas como eje de integración
- Separación de responsabilidades: UI (componentes) separada de lógica de negocio y datos (services, hooks, types)
- Enrutamiento fuertemente tipado y generado automáticamente: routeTree.gen.ts garantiza consistencia entre compilación y rutas
- UI reutilizable: VehicleCard, Header, etc. promueven composición de vistas
- Tipado centralizado: src/types mantiene coherencia de datos entre capas
- Toaster/Notificaciones: presencia de Toaster para retroalimentación de UX
- Convenciones de código: alias @ para imports, uso de TS/JSX moderno

Notas sobre hallazgos de implementación (no modificaciones):
- layout y ruta raíz: src/routes/__root.tsx define un Outlet y Toaster para un layout consistente
- LandingPage (src/pages/landingPage.tsx) orquesta UI con hooks useVehicles y useUser; manejo básico de fechas y validaciones con toast
- No se observan pruebas unitarias o E2E en los fragmentos revisados; manejo de errores de datos no es explícito en hooks analizados

## 4. Recomendaciones de mejora
1) Desacoplar la lógica de LandingPage
- Extraer la lógica de fechas, validaciones y orquestación de datos a un hook dedicado (p. ej., useLandingSearch) o a un servicio de dominio (p. ej., useVehicleFilters).
- Beneficios: mejor testabilidad, reutilización y menor acoplamiento en la página de presentación.

2) Introducir estado global cuando haga falta
- Evaluar la adopción de Context + useReducer para estados transversales (UI, usuario, filtros) a medida que la app crezca.
- Documentar qué se gestiona en contexto y cuándo migrar del approach actual de hooks locales.

3) Guardias y loaders en rutas
- Añadir guards/loaders para rutas sensibles (p. ej., publish, bookings) para validar autenticación y datos requeridos antes del render.
- Verificar si routeTree.gen.ts/ pipeline de generación soporta metadata de autenticación; evitar tocar archivos generados manualmente.

4) Robustecer manejo de datos y estados de carga/errores
- Implementar estados de carga y manejo de errores en hooks como useVehicles y useUser (spinner, skeletons, mensajes de error con retry).
- Mejorar UX ante fallos de red o respuestas erróneas.

5) Organización y escalabilidad
- Considerar una organización por features complementaria a la basada en rutas (car, user, booking, search, publish) para facilitar crecimiento y mantenimiento.
- Mantener routeTree.gen.ts fuera del control directo; asegurar pipeline CI para generación y validación.

6) Calidad y pruebas
- Añadir tests unitarios para componentes clave (e.g., LandingPage, VehicleCard) y hooks (useVehicles, useUser)
- Incluir pruebas de integración para flujos como búsqueda y navegación a /search con parámetros
- Pruebas de UI para Toaster y validaciones de UI

7) Documentación y gobernanza
- Mantener ARCHITECTURE_REPORT.md actualizado con: flujo de enrutamiento, dependencias, patrones, y recomendaciones; incluir guía sobre generación de routeTree.gen.ts
- Documentar convenciones de nomenclatura y uso de alias @ para onboarding

8) Preguntas para alinear con el equipo
- ¿Se planea introducir estado global con reducer+context? ¿qué módulos serían prioritarios?
- ¿Qué rutas deben considerarse protegidas y qué comportamiento de autenticación se espera?
- ¿Cómo se valida la generación de routeTree.gen.ts en CI y quién es responsable?

## 5. Resumen ejecutivo
- La arquitectura actual es modular y orientada a características con routing generado y tipado (TanStack Router).
- LandingPage centraliza lógica de UI y datos, pero podría beneficiarse de abstracción en hooks y mejor manejo de estados/cargas de datos.
- No se detectó uso activo de Context global; planificar adopción gradual de Reducer+Context según necesidades.
- Acciones prioritarias: refactor de LandingPage, introducción de guards/loads en rutas, mejor manejo de errores, y expansión de pruebas y documentación.

Si quieres, puedo convertir estas recomendaciones en un plan de trabajo con tareas y criterios de éxito, o ampliar este reporte con diagramas de alto nivel.