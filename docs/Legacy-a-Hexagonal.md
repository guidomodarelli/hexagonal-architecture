# 📝 Estrategia de Refactor (*Legacy* a Hexagonal)

El refactor debe ser **incremental y seguro**, garantizado mediante *tests* de aceptación que validen el comportamiento en cada paso.

## 🎯 Fases del Refactor

### 1️⃣ Añadir *Tests* de Aceptación (E2E)
Cubrir flujos críticos (crear, listar, borrar) y asegurar un entorno determinista (mocks, *seed* de datos). Estos *tests* serán tu red de seguridad durante todo el proceso.

### 2️⃣ Identificar Responsabilidades (*Seams*)
Analizar el código actual para ubicar dónde está cada responsabilidad:
- **UI**: Componentes que renderizan y capturan eventos
- **Orquestación**: Código que coordina múltiples operaciones
- **Casos de uso**: Lógica de negocio (aunque esté mezclada con UI)
- **Validaciones**: Reglas que verifican datos
- **Persistencia**: Llamadas a `localStorage`, APIs, etc.

Documenta cómo se relacionan entre sí (qué llama a qué) para identificar puntos de separación (*seams*) donde puedes insertar abstracciones.

### 3️⃣ Migración Gradual a TypeScript
Comenzar por tipos esenciales: Entidades, DTOs e *interfaces* de repositorio. Esto proporciona seguridad de tipos desde el inicio.

### 4️⃣ Definir Interfaz de Repositorio
Extraer interacciones con la fuente de datos (*localStorage*, API) a una *interface* en *domain*. Implementar el adaptador concreto en *infrastructure*.

### 5️⃣ Implementar Casos de Uso
Crear funciones puras en *application* con `dependency injection` por currying. El primer nivel recibe dependencias estables del módulo; el segundo, el input del caso de uso. Mantén la lógica agnóstica de la infraestructura.

### 6️⃣ Mover Reglas de Negocio al Dominio
Crear *value-objects* y validadores. **Principio clave:** el dominio lanza errores de negocio; los casos de uso los coordinan y propagan. La traducción a HTTP/UI ocurre en el borde.

### 7️⃣ Crear `modules/<feature>/setup.[tj]s`
Usarlo como **composition root** por feature:
- Instanciar adapters concretos
- Construir casos de uso curried
- Exportar handlers o `useCases` del módulo ya compuestos
- Dejar que `main.ts`, `index.ts` o el entrypoint del framework solo consuman el `setup` del feature que necesitan

### 8️⃣ Escribir *Tests* Unitarios
Con la lógica aislada:
- *Tests* unitarios para *domain* y *application* (usando repositorios *mock*)
- *Tests* de integración para adaptadores (*infrastructure*)

### 9️⃣ Refactorizar UI
Extraer lógica de presentación a *custom hooks* o servicios que deleguen en casos de uso. La UI debe ser lo más **delgada** posible.

### 🔟 Revisión y Optimización Continua
Revisar código refactorizado, optimizar según necesidad y **validar que los *tests* de aceptación sigan en verde**.

### 1️⃣1️⃣ Documentar Decisiones (ADRs)
Registrar decisiones de arquitectura (*Architecture Decision Records*) para mantener contexto y razonamiento.

### 1️⃣2️⃣ Iterar
Repetir el proceso para otras áreas del sistema hasta completar la migración.

`<source-root>` es `src` si el proyecto usa `src/`; si no, es la raíz del repositorio. El composition root recomendado vive en `modules/<feature>/setup.[tj]s`; si además existe un `modules/setup.[tj]s` global, mantenelo como agregador liviano o factories lazy por feature.

---

## ✅ Beneficios de este Enfoque

- ✨ **Migración segura y controlada**
- 🛡️ Minimiza riesgos al negocio
- 📊 Permite medir progreso incremental
- 🔄 Facilita *rollback* si es necesario
- 👥 Equipo puede trabajar en paralelo en diferentes capas
