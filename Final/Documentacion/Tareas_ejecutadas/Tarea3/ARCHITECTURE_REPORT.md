# Reporte de Arquitectura: PickNDrive Frontend (Frontend - React + TS)

Este documento sintetiza la arquitectura observada en el repositorio de PickNDrive-IA, enfocándose en el frontend (UI/UX) y su pipeline de construcción, ejecución y despliegue. Se apoya en los hallazgos del explorer y del researcher, y propone mejoras de cara a la escalabilidad y la mantenibilidad.

## 1) Stack tecnológico (stack principal)
- React 19 + TypeScript
- Vite como servidor de desarrollo y bundling
- Tailwind CSS v4 para estilos utilitarios
- TanStack: React Query y TanStack Router para data-fetching y enrutamiento
- Radix UI junto con ShadCN/UI para componentes UI consistentes
- Axios para llamadas a APIs
- date-fns, lucide-react, zod, etc. para utilidades y tipado
- Husky, ESLint y Prettier para calidad de código (lint/format/type-check)
- Estructuras de testing y tooling indicadas en package.json (lint, format, type-check)

> Nota: la documentación de arquitectura hace mención a un pipeline multi-agente (explorer, researcher, implementer, tester, reviewer). Este reporte representa el deliverable del subagente Implementer sobre la base de los descubrimientos disponibles.

## 2) Estructura de carpetas principal (src)
- src/
  - App.css, App.tsx, index.css, main.tsx: punto de entrada y estilos globales
  - assets: recursos estáticos (imágenes, iconos, etc.)
  - components: componentes reutilizables y módulos de UI
    - ui: componentes UI genéricos (Header, VehicleCard, etc.)
    - rent-my-car, reset-forget-password, vehicle-details-page, user-tabs: módulos UI/flujo específicos
    - ProtectedRoute: wrapper de ruta protegida
  - constants: datos estáticos (brands.ts, cities.ts)
  - hooks: hooks de negocio (useVehicle, useBooking, usePasswordReset, etc.)
  - lib: utilidades y helpers (date.ts, toast.ts, utils.ts)
  - routes: configuración de rutas a nivel de pipeline de la app
  - pages: páginas de alto nivel (About, Auth, Booking, Profile, VehicleDetails, etc.)
  - services: capa de API (api.ts, rentals.ts, trips.ts, passwordResetService.ts)
  - types: definiciones de tipos (api.ts, vehicle.ts, booking.ts, user.ts, etc.)
  - routeTree.gen.ts, router.ts: configuración de enrutamiento y generación de ruta
  - vite-env.d.ts: tipado específico de Vite
- README.md y otros ficheros de configuración en la raíz de pickndrive-ia

## 3) Convenciones y patrones observados
- Arquitectura modular basada en componentes y hooks (Single-Responsibility por carpeta de funcionalidad)
- Tipado fuerte con TypeScript; separación clara entre types y runtime
- Enrutamiento con TanStack Router y componentes de ruta en src/routes/pages
- Acceso a datos vía API wrapper en src/services y modelos en src/types
- Estilos con Tailwind y componentes UI basados en Radix UI/ShadCN
- Lint/format y tooling (eslint, prettier, husky, lint-staged) para calidad de código
- Archivos de configuración de limpieza y construcción (package.json, tsconfig*, vite.config)

## 4) Patrones y componentes clave
- UI general
  - Header.tsx, DataSelect.tsx, VehicleCard.tsx, VehicleCardDisplay.tsx, SearchNav.tsx, SortButton.tsx, etc.
  - avatar.tsx, badge.tsx, button.tsx, input.tsx, select.tsx, dialog.tsx, popover.tsx, etc.
- Funcionalidad de vehículos y reservas
  - Components: VehicleDetail, VehicleDisplayModal, VehicleAvailabilityCalendar, ReviewsSection
  - RentMyCar: Form.tsx, ui.tsx
  - Vehicles list/browsing: AvailableVehicles, VehicleInfoButton, VehiclesNumberDisplay
- Flujo de usuario y autenticación
  - ProtectedRoute, auth.tsx
  - Auth pages y Password reset (ForgotPasswordForm, ResetPasswordForm, ValidResetPasswordForm)
- Detalles y datos
  - pages: VehicleDetailsPage, BookingPage, ProfilePage, RentMyCar, AuthPage, etc.
  - hooks/services para gestión de usuarios, reservas, vehículos, imágenes
- Servicios y tipos
  - src/services/api.ts, rentals.ts, trips.ts
  - src/types para APIs, vehicle, booking, user, etc.
- Routing y configuración de pipeline
  - src/router.ts, src/routes y routeTree.gen.ts

## 5) Flujo de ejecución y enrutamiento
- Entrada de la app: main.tsx inicia React, React Query y RouterProvider
- Router central (router.ts) gestiona rutas y navegación entre páginas (landing, search, vehicle details, profile, bookings, etc.)
- Datos: React Query maneja data fetching/cache; servicios API encapsulan llamadas a back-end
- Tipos y modelos: tipos bien definidos en src/types para garantizar consistencia entre UI y API

## 6) Dependencias y flujos de integración
- Paquetes clave: react, typescript, vite, tanstack router, tanstack query, tailwind, radix ui, shadcn/ui, axios, date-fns, zod, lucide-react, husky, eslint, prettier
- Flujo de desarrollo: desarrollo local con Vite, lint/format/type-checking como parte del pipeline de CI/local
- Testing: presencia de scripts de lint/format/type-check; se recomienda expandir con tests unitarios de hooks, tests de UI y pruebas de flujo.

## 7) Riesgos y mitigaciones
- Gestión de estado para flujos complejos (multi-step): actualmente manejo local en Form.tsx. Riesgo: dificultad de mantenimiento y propagación de cambios.
  - Mitigación: introducir un FormContext/reducer dedicado para RentMyCar con un origen de verdad y acciones claras (SET_ATTRIBUTE, NEXT_STEP, etc.).
- Organización de código y wiring: riesgo de crecimiento desordenado a medida que el proyecto escala.
  - Mitigación: agrupar por dominio (por ejemplo, src/features/rentMyCar) y usar barrel exports para simplificar imports. Code-splitting vía rutas lazy para módulos pesados.
- Carga de datos y errores: posibilidad de no manejar correctamente loading states o errores en rutas complejas.
  - Mitigación: usar loaders/actions de TanStack Router (donde sea posible) con Suspense y boundary de error.
- Accesibilidad y pruebas: componentes UI personalizados podrían carecer de accesibilidad y pruebas.
  - Mitigación: agregar roles ARIA, labels y foco gestionado; ampliar pruebas unitarias e de integración para flujos críticos.
- Consistencia de tipado: usos de cualquier y aserciones no verificadas podrían causar problemas.
  - Mitigación: reforzar typing de props y funciones, revisar IDs de inputs para evitar duplicados.
- Internacionalización: falta de i18n centralizado.
  - Mitigación: planificar un sistema de strings centralizadas para futuro i18n.

## 8) Comandos relevantes (desarrollo, buildu y testing)
- Instalación e instalación de dependencias
  - yarn install o npm install
- Desarrollo local
  - yarn dev o npm run dev
- Lint y formato
  - yarn lint o npm run lint
  - yarn format o npm run format
- Type checks
  - yarn typecheck o npm run type-check
- Build y empaquetado
  - yarn build o npm run build
- Pruebas (sugerencia para ampliación)
  - yarn test o npm run test
- Verificación rápida de tipos sin compilación
  - tsc --noEmit

## 9) Patrones de arquitectura observados
- Modularidad basada en dominios y componentes: cada carpeta maneja una parte de la UI o flujo (vehículos, auth, rentar, detalles).
- Tipado fuerte: TypeScript con separación entre types y runtime, reducción de errores de integración entre UI y datos.
- Enrutamiento escalable: TanStack Router con routeTree.gen.ts y router.ts para generar rutas de forma consistente y escalable.
- Capa de datos: servicios/API wrappers centralizados (src/services) y modelos en src/types para garantizar contrato entre UI y back-end.
- UI y estilo: Tailwind + Radix UI y una UI library propia (ShadCN) para consistencia visual y experiencia de usuario.
- Calidad de código: husky + eslint + prettier + lint-staged para asegurar código limpio en commits.

## 10) Recomendaciones de mejora (plan de acción)
1) Centralizar estado de formularios multi-etapa
   - Crear RentMyCarFormContext con un reducer para manejar atributos, pasos y validaciones. Reutilizar componentes de UI alimentados por este contexto.
2) Organización por dominio y code-splitting
   - Crear src/features/rentMyCar/ con subcomponentes, hooks y tipos; usar exports barrel y rutas lazy para rutas pesadas.
3) Aprovechar TanStack Router para data y acciones
   - Explorar loaders y actions; envolver rutas en Suspense y boundaries; definir rutas con pre-carga de datos cuando aplique.
4) Consistencia y tipado
   - Añadir tipos explícitos para props de UI; revisar data inputs para evitar IDs duplicados; evitar any y aserciones inseguras.
5) Accesibilidad y pruebas
   - Añadir labels ARIA, navegación por teclado y pruebas de accesibilidad básica; ampliar suite de pruebas: unitarias para hooks y pruebas de integración para flujos críticos.
6) Internacionalización y mensajes
   - Diseñar una configuración central de textos para facilitar futura i18n (archivos de constantes o i18n provider).
7) Observaciones de implementación (prácticas existentes)
   - Revisar el uso de non-null assertions en CategoryButton; evaluar robustez de data source cuando sea undefined.
8) Cómo empezar la próxima iniciativa de refactor
   - Propuesta: plan de refactor en tres fases: (a) scaffolding del FormContext; (b) reorganización por dominio; (c) introducción de loaders/ Suspense en rutas; (d) incremento de tests.

## Anexo: notas y referencias
- Este reporte se apoya en los hallazgos del explorer y researcher de este workspace.
- Existe un documento ARQUITECTURA.md que describe un pipeline multi-agente para analizar repos y generar informes; ARCHITECTURE_REPORT.md aún no está presente y se genera aquí para cumplir con el flujo de trabajo.

## Conclusión
El frontend de PickNDrive-IA está bien posicionado para escalar gracias a una pila moderna, una estructura de carpetas clara y patrones de diseño consistentes. Las áreas de mejora identificadas buscan estabilizar flujos complejos (especialmente RentMyCar), mejorar la organización del código y fortalecer la base de pruebas y accesibilidad para una entrega de mayor calidad.
