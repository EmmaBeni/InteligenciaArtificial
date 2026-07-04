# Caso de Uso

## Respositorio
Recurrimos en este caso de uso al repositorio de frontend de PickNDrive, desarrollado por algunos de nosotros en el transcurso de Laboratorio II.
El mismo utiliza componentes de React. 

    https://github.com/emmabeni27/pickndrive-ia

Originalmente usamos como prueba:

    https://github.com/facebook/create-react-app

## Objetivo
Con la arquitectura de Explorer + Researcher + Implementer + Tester + Reviewer el objetivo es:

- Explorer: mapea estructura, stack, convenciones. Analiza el repositorio.
- Researcher: usa RWAG + web para revisar si las versiones de las dependencias encontradas por 
Explorer están deprecadas, tienen CVEs conocidos o si hay uan versión más nueva recomendada. Le da propósito
al uso del RAG.
- Implementer: redacta el reporte final con las recomendaciones. 
- Tester: valida que el reporte tenga las secciones mínimas (stack, dependencias, riesgos, recomendación).
- Reviewer: revisa que el reporte responde el pedido original y no se haya desviado. 

## Criterio de Cumplimiento
El reporte se considera exitoso si:

* Identifica correctamente el stack real del proyecto (framework, versión, bundler, gestor de paquetes).
* Lista las dependencias del package.json con sus versiones, y señala cuáles están desactualizadas respecto a la versión estable actual (esto obliga a que el Researcher haga una búsqueda real).
* Señala al menos 2-3 riesgos concretos - no bastan observaciones vagas.
* El Reviewer aprueba el reporte como alineado al pedido original (traza donde compara el resultado contra original_request y evalúa).