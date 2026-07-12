# Informe de Arquitectura - frontend de PickNDrive (pickndrive-ia)

Este documento sintetiza la arquitectura observada en el repositorio frontend, con hallazgos derivados del analysis (explorer) y buenas prácticas de diseño (researcher). Sirve como guía para mantenimiento, escalabilidad y mejoras.

## 1) Stack tecnológico (Tech Stack)
- Frontend
  - Lenguaje: TypeScript
  - Framework: React
  - Buildeo y tooling: Vite
  - Enrutamiento: TanStack Router (routeTree.gen.ts genera rutas)
  - Estilo: Tailwind CSS
  - Lint/Formato: ESLint y Prettier
  - Consumo API: Axios (patrón de servicio con interceptores)
- Backend (observación del patrón API)
  - Backend tipo Strapi o similar, endpoints REST con populate (ej. /auth/local, /vehicles, /bookings, /reviews, /upload)
  - Respuestas anidadas con data.data, estructuras con populate para relaciones

## 2) Estructura de carpetas (principal)
- src/pages
  - Contiene las pantallas principales (AboutPage, AuthPage, BookingPage, VehicleDetailsPage, etc.)
  - Cada página contribuye a las rutas expuestas por routeTree.gen.ts
- src/components
  - Componentes reutilizables (ProtectedRoute, UI primitives, RentMyCar, etc.)
- src/hooks
  - Hooks de negocio (p. ej. useVehicle, useBooking, useAddAttributesToVehicle, etc.)
- src/services
  - Capa de acceso a la API, agrupada por dominio:
    - api.ts: instancia de Axios, baseURL dinámico (VITE_API_BASE_URL) y interceptor de JWT desde localStorage
    - apiService: autenticación y usuario (signIn, signUp, getCurrentUser, updateCurrentUser, getUserStats, getAllVehicles)
    - bookingService: createBooking, getBookingById, getUserBookings
    - createVehicleService: getVehicleById, createVehicle, updateVehicle, deleteVehicle
    - vehicleService: getVehicle, getAllCategories, getAttributesByCategory, getAllFeaturesByCategory, getOptionsByAttribute, getBookingsFromVehicle, createVehicle, updateVehicle, addAttributesToVehicle, addFeaturesToVehicle, uploadImages, getVehicleAvailability
    - reviewService: createReview, getUserReviews
- src/types
  - Definiciones de tipos/interfaces: api.ts, booking.ts, review.ts, user.ts, vehicle-data.ts, vehicle.ts
- src/constants / src/lib
  - Constantes y utilidades/ helpers (estructura indicada, usos en hooks/components)
- src/services (archivo central de servicios) y rutas de datos
  - Patrones de datos poblados (populate) y consultas con filtros para obtener relaciones (vehículos, categorías, atributos, reseñas, imágenes, etc.)
  - Uso de FormData para subir imágenes

## 3) Patrones y decisiones de arquitectura
- Capas y modularidad
  - Capa de servicios centralizada por dominio (auth/users, vehicles, bookings, reviews) con una API centralizada (Axios) y un interceptor para JWT.
  - Tipos fuertemente tipados en TypeScript para usuarios, vehículos, reservas, reseñas, etc.
- Enrutamiento y vistas
  - TanStack Router genera las rutas a partir de pages mediante routeTree.gen.ts.
  - Páginas separadas para gestión de permisos (ProtectedRoute) y UI consistente.
- Modelo de datos y relaciones
  - Relaciones anidadas pobladas mediante query strings (populate) para obtener información relacionada de vehículos, categorías, atributos, imágenes, reseñas, etc.
  - Mecanismo de subida de assets (imágenes) vía FormData.
- Estado y flujo de datos
  - Patrón Reducer + Context para estado global (según hallazgos del researcher): un par de contextos (state y dispatch) para cada dominio o para el estado global con slices.
  - AppProviders como orquestador de providers (AuthProvider, VehicleProvider, BookingProvider, etc.) para evitar anidaciones profundas.
- Persistencia y seguridad
  - JWT almacenado en localStorage; interceptor de Axios para adjuntar token en cada request.
- Observabilidad y documentación
  - Observabilidad mencionada como posible future integration (Langfuse), pero la doc actual se centra en el frontend. Se recomienda documentar decisiones de arquitectura React en un archivo dedicado.
- Rendimiento
  - Potencial uso de code-splitting con React.lazy/Suspense para rutas y componentes pesados.

## 4) Dependencias clave (versión conceptual)
- React, TypeScript, Vite, TanStack Router, Tailwind CSS, Axios, ESLint, Prettier
- Otras utilidades/typing: types/api.ts, types/vehicle.ts, etc.

## 5) Flujo de datos (alto nivel)
- Usuario realiza una acción en la UI -> llama a un servicio (ex: bookingService.createBooking) -> servicio compone payload y llama API (Axios) -> API responde con datos en estructura anidada (populate) -> los datos se transforman si es necesario en modelos TypeScript y se almacenan/emitidos al UI vía reducers/contexts -> UI renderiza.

## 6) Modelo de datos y relaciones (conceptual)
- User, Vehicle, Booking, Review, Category, Attribute, Feature, Image
- Relaciones a través de populate (ej.: Vehicle con Category, Attributes, Features, Bookings, Reviews, Image list)
- Uploads de imágenes vía multipart/form-data para vehicles

## 7) Roles, permisos y protección de rutas
- ProtectedRoute controla el acceso a rutas dependientes de autenticación.
- Estados de autenticación gestionados por context/slices, con guardado de usuario y tokens en localStorage.

## 8) Comandos relevantes (entorno de desarrollo)
- Instalación y arranque
  - npm install o yarn install
  - npm run dev (o npm run start) para arrancar Vite
- Lint y Formateo
  - npm run lint
  - npm run format
- Construcción y pruebas
  - npm run build
  - npm test (si hay configuración de tests)
- Exploración y diagnóstico del repo
  - ls -la
  - ls -R src
  - grep -R "routeTree.gen.ts" -n
  - rg "ProtectedRoute|useReducer|Context" -n src
- Inspección de API/servicios
  - grep -R "axios.create" -n src/services
  - sed -n '1,120p' src/services/api.ts

## 9) Patrones usados y buenas prácticas
- Servicios por dominio con API centralizada (Axios + interceptores)
- Tipos TypeScript para robustez estática
- Enrutamiento dinámico y rutas protegidas (TanStack Router, ProtectedRoute)
- Capa de UI/UX consistente con Tailwind
- Arquitectura de estado basado en Reducer + Context (posible uso de AppProviders)
- Relaciones anidadas a través de populate en llamadas REST
- Subida de imágenes con FormData
- Código modular y separación por dominios

## 10) Riesgos y mitigación
- Riesgo: dispersión del wiring (context/reducer repartidos entre archivos)
  - Mitigación: centralizar en un AppProvider con wiring consistente; crear archivos de wiring por dominio y/o un patrón de modules/features.
- Riesgo: dependencias a la API backend (Strapi-like) con populate; cambios en la API pueden romper el frontend.
  - Mitigación: definir y documentar los endpoints esperados, usar tipos robustos y pruebas de contrato.
- Riesgo: token en localStorage (vulnerabilidad XSS) y ataques 
  - Mitigación: analizar opciones de seguridad (Refresh tokens, httpOnly cookies) y habilidad para invalidar sesiones; validar token en interceptor.
- Riesgo: dimensionamiento de data en populate que puede generar payloads grandes
  - Mitigación: usar lazy loading de relaciones y limitar populate a lo necesario; paginación para grandes collections.
- Riesgo: rendimiento y bundle size si no se usa code-splitting
  - Mitigación: aplicar lazy loading a rutas/pequeños módulos grandes y revisar bundles con herramientas de análisis.
- Riesgo de mantenimiento por documentación desalineada
  - Mitigación: crear ARCHITECTURE_REACT.md específico y mantenerlo actualizado; enlazarlo en README.
- Riesgo de pruebas insuficientes
  - Mitigación: añadir tests de reducers y tests de providers; pruebas de integración para flujos clave (login, creación de vehicle, etc.)

## 11) Recomendaciones de mejora (acciones concretas)
1) Estandarizar pattern de reducer + contexto
   - Crear DomainStateContext y DomainDispatchContext por dominio o una solución central con slices.
   - Implementar Action discriminated unions en TS y usar useReducer.
2) Centralizar wiring en AppProviders
   - Un único punto para envolver la app con AuthProvider, VehicleProvider, BookingProvider, etc.
   - Reducir la duplicación de wiring en componentes/pages.
3) Organización basada en features/dominios
   - Reorganizar src en src/features/Auth, src/features/Vehicles, src/features/Bookings, etc., con subcarpetas hooks/components/pages/coherentes.
4) Code splitting y lazy loading
   - Implementar React.lazy para páginas pesadas y rutas; usar Suspense con fallback.
5) Documentación de arquitectura React
   - Crear ARCHITECTURE_REACT.md con decisiones técnicas, reglas y diagramas.
6) Mejora de tipado y seguridad
   - Asegurar discriminated unions para actions; genéricos en contexts; revisar manejo de tokens y autenticación.
7) Testing de la capa de estado
   - Añadir tests de reducers, tests de providers; pruebas de integración para flujos clave.
8) Accesibilidad y UX de routing
   - Asegurar roles/aria-labels en ProtectedRoute; notificaciones accesibles.
9) Observabilidad
   - Registrar logs/telemetría para flujos críticos; evaluar integración con Langfuse o similares.
10) Plan de migración progresivo
   - Si se quiere migrar, hacerlo en incrementos pequeños (p. ej., primer dominio Vehicle, luego Auth, etc.).

## 12) Notas finales
- El stack y la estructura mostrados en este informe están alineados con los hallazgos del explorer y researcher provistos en la documentación del repositorio. Se recomienda mantener una arquitectura local de frontend bien documentada para evitar pérdidas de contexto, especialmente cuando el equipo crece o cuando se integran nuevos módulos.

## Anexo: diagrama conceptual (alto nivel)
- UI (React + TS) -> AppProviders (Auth, Vehicle, Booking, etc.) -> Contexts/Reducers -> API Services (Axios) -> Backend Strapi-like
- Enrutamiento: TanStack Router genera rutas desde routeTree.gen.ts; ProtectedRoute protege vistas basadas en estado de autenticación.

"ARCHITECTURE_REPORT" generado automáticamente a partir de hallazgos de exploración y recomendaciones de diseño.