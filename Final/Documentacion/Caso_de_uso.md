# Caso de Uso

## Respositorio
Recurrimos en este caso de uso al repositorio de frontend de PickNDrive, desarrollado por algunos de nosotros en el transcurso de Laboratorio II.
El mismo utiliza componentes de React. 

    https://github.com/emmabeni27/pickndrive-ia

Originalmente usamos como prueba:

    https://github.com/facebook/create-react-app

## Objetivo
Con la arquitectura de Explorer + Researcher + Implementer + Tester + Reviewer el objetivo es:

- **Explorer**: Mapea la estructura del repositorio, stack tecnológico, dependencias y convenciones de código.
- **Researcher**: Utiliza el RAG (con documentación indexada de React) y búsquedas web para investigar buenas prácticas y comprobar si las dependencias encontradas están obsoletas, tienen vulnerabilidades (CVEs) o si existen versiones recomendadas estables.
- **Implementer**: Redacta el reporte final de arquitectura (`ARCHITECTURE_REPORT.md`) con hallazgos y recomendaciones.
- **Tester**: Valida que el reporte haya sido generado, no esté vacío y contenga las secciones clave de arquitectura.
- **Reviewer**: Realiza la revisión final del reporte contra la solicitud original del usuario (`original_request`) y dictamina su aprobación o rechazo con justificación. 

## Criterio de Cumplimiento
El reporte se considera exitoso si:

* Identifica correctamente el stack real del proyecto (framework, versión, bundler, gestor de paquetes).
* Lista las dependencias del package.json con sus versiones, y señala cuáles están desactualizadas respecto a la versión estable actual (esto obliga a que el Researcher haga una búsqueda real).
* Señala al menos 2-3 riesgos concretos - no bastan observaciones vagas.
* El Reviewer aprueba el reporte como alineado al pedido original (traza donde compara el resultado contra original_request y evalúa).

Los datos del reporte deberían concordar con aspectos como los siguientes:

Stack real del proyecto:
  * Framework principal: React v19.1.1 (usando la API moderna de React 19).
  * Enrutado (Routing):  @tanstack/react-router  v1.131.2 (enrutado basado en archivos).
  * Gestión de Estado / Fetching:  @tanstack/react-query  v5.84.2.
  * Estilos (CSS): Tailwind CSS v4.1.11 (utilizando la integración nativa a través de  @tailwindcss/vite ).
  * Compilador / Empaquetador (Bundler): Vite v7.1.0 con TypeScript (~v5.8.3).
  * Gestor de paquetes: pnpm (identificado por la presencia de pnpm-lock.yaml y pnpm-workspace.yaml).

Riesgos:
  * Fuga de sesión y datos en caché en el logout. La función dedicada no limpia ni invalida caché global.
  * Pérdida de Reactividad al leer credenciales alojadas en localStorage.
  * Fragilidad en el procesamiento de fechas y cálculo de duración.