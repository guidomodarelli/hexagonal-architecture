## Añade Arquitectura Hexagonal a un proyecto existente

### Plan de refactor: de main.js a arquitectura hexagonal

**Objetivo:** Extraer la lógica de negocio, introducir TypeScript y aplicar separación de capas manteniendo la aplicación funcional mediante tests de aceptación.

---

#### 1. Añadir tests de aceptación (E2E)

- Implementar pruebas que cubran los flujos críticos (crear curso, listar, borrar) con **Playwright** o **Cypress**.
- Asegurar un entorno determinista (seed de localStorage, mocks de red).
- Usar estos tests como **guardia durante el refactor**: la aplicación debe seguir pasando las E2E en cada cambio.

#### 2. Identificar seams y responsabilidades en main.js

- Mapear responsabilidades: UI, orquestación, casos de uso, validaciones, persistencia.
- Definir puntos de extracción (funciones/objetos que pueden recibir dependencias).

#### 3. Migración incremental a TypeScript

- Activar `allowJs` en `tsconfig.json` y renombrar gradualmente archivos a `.ts`/`.tsx`.
- Migrar primero tipos esenciales (`Course`, DTOs, interfaces de repositorio) para ganar seguridad de tipo sin bloquear el desarrollo.

#### 4. Definir interfaz de repositorio (infraestructura)

Extraer las interacciones con localStorage a una interfaz en `domain`/`application`:

```typescript
export interface CourseRepository {
  save(course: Course): Promise<void>;
  findAll(): Promise<Course[]>;
  findById(id: string): Promise<Course | null>;
  delete(id: string): Promise<void>;
}
```

Proveer una implementación concreta (`LocalStorageCourseRepository`) en `infrastructure/`.

#### 5. Implementar casos de uso en la capa de aplicación

Crear funciones puras con `dependency injection` por currying. El primer nivel recibe dependencias estables; el segundo, el input del caso de uso:

```typescript
type CreateCourseDependencies = {
  courseRepository: CourseRepository;
  generateId: () => string;
};

export const CreateCourse =
  ({ courseRepository, generateId }: CreateCourseDependencies) =>
  async (req: CreateCourseRequest): Promise<void> => {
    const course: Course = {
      ...req,
      id: generateId(),
    };

    // Lógica del caso de uso
    await courseRepository.save(course);
  };
```

Mantener la orquestación y la lógica de negocio en `use-cases`; la UI sólo invoca estos use-cases. Los adapters concretos de `infrastructure` pueden ser clases (`LocalStorageCourseRepository`, `CourseRepositoryHttp`, etc.) y se instancian desde `setup.[tj]s`.

#### 6. Mover reglas de negocio al dominio

- Crear **value-objects** y validadores (`CourseTitle`, `CourseDuration`, `CourseId`, `ensureCourseIsValid`).
- Lanzar errores de dominio desde el dominio; los use-cases los coordinan y propagan. La traducción al contrato externo (HTTP/UI) ocurre en el entrypoint.

#### 7. Crear `modules/<feature>/setup.[tj]s` como builder y usar `modules/setup.[tj]s` solo si conviene centralizar el wiring

- Crear `<source-root>/modules/<feature>/setup.[tj]s` por feature para declarar `build<Feature>Module(deps)` con dependencias explícitas y devolver `useCases` ya compuestos, sin instanciar adapters concretos.
- Si conviene centralizar dependencias compartidas, crear `<source-root>/modules/setup.[tj]s` como agregador o factories lazy por feature y exponer `createRequestModules()`.
- Si el framework exige un `main.ts`, `index.ts` o archivo de routing, ese entrypoint puede **consumir** `createRequestModules()` o importar solo `build<Feature>Module(deps)` para componer el feature que necesita.
- Evitar lógica de negocio y acceso directo a localStorage en el entrypoint del framework.

#### 8. Escribir tests unitarios una vez aislada la lógica

- Tests unitarios para `domain` y `application` (use-cases) usando repositorios mock.
- Tests de integración para adaptadores (`LocalStorageCourseRepository`).

---

### Checklist rápido

- [ ] E2E cubriendo flujos críticos en el código legacy
- [ ] Interfaz `CourseRepository` y adaptación a localStorage
- [ ] Use-cases puros con `dependency injection` por currying
- [ ] Domain: value-objects y validadores
- [ ] `<source-root>/modules/<feature>/setup.[tj]s` creado como builder por feature con `build<Feature>Module(deps)`
- [ ] Si hace falta wiring compartido, `<source-root>/modules/setup.[tj]s` agregado como composition root opcional
- [ ] El entrypoint del framework reducido a entorno/arranque y composición del feature vía `createRequestModules()` o `build<Feature>Module(deps)`
- [ ] Migración completa a TypeScript y suite de tests unitarios funcionando

---

`<source-root>` es `src` si el proyecto ya usa `src/`; si no, es la raíz del repositorio.

**Resultado esperado:** Código desacoplado, testable y preparado para futuros cambios de infraestructura o framework sin tocar la lógica de negocio.
