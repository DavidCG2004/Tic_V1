







# Análisis Completo: Estado del Backend vs. Plan V2

**Fecha:** 10 de octubre de 2026  
**Proyecto:** Sistema de Rutas de Aprendizaje EPN (ESFOT)  
**Analista:** Ingeniero Senior Software (>10 años experiencia)

---

## 1. Veredicto del Plan V2: Sólido pero con brechas no cubiertas

El `PLAN-GRAFO-CONOCIMIENTO_V2.md` es un plan bien estructurado que **sí solventa las brechas principales** (HU-013, HU-014, HU-015 y motor de grafo), pero deja varias brechas técnicas sin abordar.

---

## 2. Brechas que el Plan V2 SÍ cubre

| Brecha (AppContexto.md) | Solución en Plan V2 | Estado |
|--------------------------|---------------------|--------|
| Autodiagnóstico (HU-013) | Fase 2: endpoints + `SelfDiagnosisModal.jsx` | Cubierto |
| Ruta personalizada (HU-014) | Fase 3: `graphAlgorithm.js` con graphlib + orden topológico | Cubierto |
| Chat con GraphRAG (HU-015) | Fase 4: subgrafo inyectado a Gemini | Cubierto |
| Recomendaciones Neo4j | Sección 5: `GET /api/graph/recommendations` | Cubierto |
| Seeds piloto | Fase 1: `seedRedes.js` + `seedAplicacionesWeb.js` | Cubierto |
| Modelo de datos topic-centric | Sección 4: Subject→Module→Topic con PREREQUISITE | Cubierto |
| `graphlib` sin usar | Sección 6: algoritmo topológico con graphlib | Cubierto |

---

## 3. Brechas que el Plan V2 NO cubre

### 3.1 Críticas (afectan producción)

| Brecha | Impacto | Plan V2 |
|--------|---------|---------|
| **Sin tests automatizados** | Riesgo de regresiones en algoritmo topológico | Mencionado en Fase 5 pero sin estrategia concreta (unit tests de graphlib, integración Neo4j mock) |
| **Sin paginación** | Los endpoints `getUsers`, `getResources`, `getRoutes` cargan todo en memoria | No mencionado |
| **Sin rate-limiting para IA** | Endpoints `chat/send`, `chat/assessment`, `routes/generate` son costosos (Gemini API) | `express-rate-limit` está instalado pero sin configuración específica para estos endpoints |
| **`.env.example` no existe** | El plan menciona su creación en Fase 0 como "completada" pero NO existe en el repo | Fase 0 marcada como completada sin evidencia |
| **`api/authService.js` roto** | Importa `./apiClient` inexistente | No mencionado |

### 3.2 Deuda técnica (afecta calidad)

| Brecha | Impacto | Plan V2 |
|--------|---------|---------|
| **`bcryptjs` duplicada** | Confusión, bundle innecesario | No mencionado |
| **`docx` sin usar** | Bundle innecesario (~2MB) | No mencionado |
| **Tabla `Permission` sin usar** | Schema muerto, confunde | No mencionado |
| **`getRecommendations` código muerto** | Engaña sobre capacidad del grafo | Se reescribe en Fase 4 pero no se limpia el stub |
| **Chat sin capa service** | Toda la lógica de Gemini en controller (252 líneas) | No se refactoriza en el plan |
| **Sin migración de datos existentes** | Los módulos actuales no tienen Topic ni PREREQUISITE | No hay plan de backfill |

### 3.3 Brechas de escalabilidad (RAG)

| Brecha | Impacto | Plan V2 |
|--------|---------|---------|
| **Sin chunking de documentos** | Contexto de Gemini se desborda con PDFs grandes | Mencionado como "no bloqueable MVP" pero sin plan de mejora |
| **Sin embeddings ni vector store** | Búsqueda semántica imposible | No mencionado |

---

## 4. Inconsistencias entre documentos y código real

| Afirmación | Realidad |
|------------|----------|
| AppContexto: "GraphRAG en chat: Parcial" | Correcto — `chat.controller.js:92-111` inyecta lista de rutas como texto |
| AppContexto: "`getRecommendations` nunca se llama" | Correcto — cero llamadas en todo el codebase |
| AppContexto: "graphlib declarada, nunca importada" | Correcto |
| AppContexto: "Sin tests" | Correcto — Jest declarado, 0 archivos |
| AppContexto: "docx declarada, nunca importada" | Correcto |
| Plan V2 Fase 0: "Completada" | **Parcialmente falso** — no existe `.env.example` en el repo |
| Plan V2: "4 agentes especializados" | Es una propuesta organizativa, no refleja equipo real |

---

## 5. Evaluación del algoritmo propuesto (Sección 6 del Plan V2)

El algoritmo de orden topológico ponderado es **correcto en concepto** pero tiene riesgos:

1. **Manejo de ciclos**: El plan usa `graphlib.isAcyclic()` pero no define qué hacer si hay ciclos (solo "registrar advertencia y desarmar"). Necesita estrategia de desarme específica.
2. **Persistencia dual Turso/Neo4j**: Crea complejidad de consistencia. Si falla Neo4j después de Turso, hay datos huérfanos. Necesita transacción compensatoria.
3. **Fórmula de prioridad**: `(6 - selfReportedLevel) * 10 + dependientes * 5 + (struggles ? 15 : 0)` es razonable pero **no está justificada pedagógicamente**. Un tema con autodiagnóstico 1 (sin conocimiento) obtiene prioridad 50, pero si es un tema terminal sin dependientes, podría ser menos urgente que un prerrequisito HARD.

---

## 6. Inventario completo del codebase

### 6.1 Estructura de archivos (27 archivos en src/)

```
src/
├── server.js                          (68 líneas)
├── api/
│   └── authService.js                 (26 líneas, ROTO - importa ./apiClient inexistente)
├── config/
│   └── turso.js                       (15 líneas)
├── controllers/
│   ├── admin.controller.js            (151 líneas)
│   ├── auth.controller.js             (523 líneas)
│   ├── chat.controller.js             (252 líneas)
│   ├── execute.controller.js          (48 líneas)
│   ├── resource.controller.js         (78 líneas)
│   ├── routes.controller.js           (443 líneas)
│   └── upload.controller.js         (112 líneas)
├── db/
│   ├── migrate.js                     (53 líneas)
│   └── schema.sql                     (198 líneas)
├── middlewares/
│   ├── auth.middleware.js             (72 líneas)
│   └── upload.middleware.js           (8 líneas)
├── routes/
│   ├── admin.routes.js                (15 líneas)
│   ├── auth.routes.js                 (28 líneas)
│   ├── chat.routes.js                 (10 líneas)
│   ├── execute.routes.js              (9 líneas)
│   ├── resource.routes.js             (12 líneas)
│   ├── route.routes.js                (28 líneas)
│   └── upload.routes.js             (13 líneas)
└── services/
    ├── auth.service.js                (81 líneas)
    ├── cloudinary.service.js         (21 líneas)
    ├── email.service.js              (151 líneas)
    ├── init.service.js               (46 líneas)
    ├── neo4j.service.js              (142 líneas)
    └── resourceParser.service.js     (60 líneas)
```

### 6.2 Estado por categoría

| Categoría | Implementado | Stub | Ausente |
|-----------|-------------|------|---------|
| Controllers | 7/7 | 0 | 1 (graph) |
| Routes | 7 módulos | 0 | 1 (graph.routes.js) |
| Services | 6/6 | 0 | 2 (knowledgeGraph, graphAlgorithm) |
| Seeds | 0 | 0 | 2 (Redes, AplicacionesWeb) |
| Tests | 0 | 0 | Todo |

### 6.3 Endpoints existentes

| Módulo | Endpoints | Estado |
|--------|-----------|--------|
| auth | 10 | 8 OK + 2 stubs inline |
| admin | 5 | Todos OK |
| resources | 4 | Todos OK |
| routes | 8 | Todos OK |
| chat | 2 | Todos OK |
| execute | 1 | OK |
| upload | 3 | Todos OK |
| **graph** | **0** | **No existe** |

---

## 7. Dependencias declaradas vs usadas

| Dependencia | Usada? | Notas |
|-------------|--------|-------|
| `@google/generative-ai` | Sí | chat.controller.js, routes.controller.js |
| `@libsql/client` | Sí | Todos los controllers |
| `axios` | Sí | resourceParser.service.js |
| `bcrypt` | Sí | auth, admin, init |
| `bcryptjs` | **NO** | Duplicada de bcrypt |
| `cheerio` | Sí | resourceParser.service.js |
| `cloudinary` | Sí | cloudinary.service.js |
| `cors` | Sí | server.js |
| `docx` | **NO** | Sin uso |
| `dotenv` | Sí | Múltiples archivos |
| `express` | Sí | Todos los routes |
| `express-rate-limit` | Sí | server.js (genérico) |
| `graphlib` | **NO** | Planeada para algoritmo topológico |
| `helmet` | Sí | server.js |
| `jsonwebtoken` | Sí | auth.middleware.js |
| `multer` | Sí | upload |
| `neo4j-driver` | Sí | neo4j.service.js |
| `nodemailer` | Sí | email.service.js |
| `pdf-parse` | Sí | resourceParser.service.js |

---

## 8. Recomendaciones ejecutivas

### Antes de ejecutar el Plan V2:

1. **Crear `.env.example`** (Fase 0 no está completa)
2. **Eliminar dependencias muertas**: `bcryptjs`, `docx` (dejar `graphlib` porque el plan lo usa)
3. **Eliminar `api/authService.js`** o marcarlo como legacy y excluirlo del build
4. **Agregar rate-limiting específico** para endpoints de IA antes de desplegar
5. **Implementar paginación** en `getUsers`, `getResources`, `getRoutes` desde el inicio

### Durante la ejecución del Plan V2:

6. **Tests del algoritmo primero (TDD)**: El algoritmo topológico es la pieza más compleja. Escribir tests antes de implementar:
   - Grafo acíclico simple
   - Grafo con prerrequisitos HARD
   - Grafo con ciclos (debe desarmar)
   - Grafo desconectado
7. **Migración de datos existentes**: Plan para convertir módulos actuales en topics con prerrequisitos
8. **Refactorizar chat**: Extraer lógica de Gemini a `chat.service.js` antes de agregar GraphRAG

### Después del Plan V2 (mejoras de calidad):

9. **Chunking + embeddings**: Para RAG escalable (Pinecone, pgvector o Chroma)
10. **CI/CD con tests**: Jest + GitHub Actions
11. **OpenAPI/Swagger**: Documentar los 35 endpoints existentes + los nuevos

---

## 9. Conclusión

| Aspecto | Veredicto |
|---------|-----------|
| ¿El Plan V2 solventa las brechas críticas (HU-013, HU-014, HU-015)? | **Sí** |
| ¿El Plan V2 está listo para ejecutarse sin ajustes? | **No** — necesita correcciones menores (Fase 0, rate-limiting, paginación, tests) |
| ¿El Plan V2 cubre toda la deuda técnica? | **No** — deja RAG, paginación, consistencia dual y refactor de chat para "después" |
| ¿Vale la pena ejecutar el Plan V2? | **Sí** — es la hoja de ruta correcta, pero debe ejecutarse con las correcciones de esta auditoría |

---

## 10. Plan de acción recomendado (ordenado por prioridad)

| Prioridad | Acción | Esfuerzo |
|-----------|--------|----------|
| P0 | Crear `.env.example` | 30 min |
| P0 | Eliminar `bcryptjs` y `docx` | 15 min |
| P0 | Eliminar/marcar `api/authService.js` como legacy | 15 min |
| P0 | Rate-limiting específico para endpoints de IA | 1h |
| P1 | Paginación en listados | 2h |
| P1 | Tests unitarios para algoritmo topológico | 4h |
| P1 | Fase 1 del Plan V2: Neo4j constraints + seeds | 1-2 días |
| P2 | Fase 2 del Plan V2: Autodiagnóstico | 3-4 días |
| P2 | Fase 3 del Plan V2: Rutas adaptativas | 5-7 días |
| P3 | Fase 4 del Plan V2: GraphRAG en chat | 1-2 días |
| P3 | Migración de datos existentes (módulos → topics) | 2-3 días |
| P3 | Refactor de chat a service layer | 1 día |
| P4 | Fase 5 del Plan V2: QA completo | 2-3 días |
| P4 | Chunking + embeddings para RAG escalable | 3-5 días |

---

*Fin del análisis. Documento generado para respaldar la toma de decisiones técnicas del proyecto ESFOT.*
