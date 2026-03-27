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

Crear funciones puras que reciban la interfaz del repositorio (inyección de dependencias):

```typescript
export function CreateCourse(repo: CourseRepository) {
  return async (req: CreateCourseRequest): Promise<void> => {
    // Lógica del caso de uso
  };
}
```

Mantener la orquestación y la lógica de negocio en `use-cases`; la UI sólo invoca estos use-cases.

#### 6. Mover reglas de negocio al dominio

- Crear **value-objects** y validadores (`CourseTitle`, `CourseDuration`, `CourseId`, `ensureCourseIsValid`).
- Lanzar errores de dominio desde el dominio; los use-cases los coordinan y propagan. La traducción al contrato externo (HTTP/UI) ocurre en el entrypoint.

#### 7. Crear `modules/<feature>/setup.[tj]s` como composition root

- Crear `<source-root>/modules/<feature>/setup.[tj]s` por feature para instanciar repositorios, construir use-cases y exportar handlers o `useCases` ya compuestos.
- Si el framework exige un `main.ts`, `index.ts` o archivo de routing, ese entrypoint debe **consumir** solo el `setup` del feature que necesita. En apps con code splitting por ruta, cada ruta importa su propio `setup`; un `modules/setup.[tj]s` global es opcional como agregador liviano o factories lazy por feature.
- Evitar lógica de negocio y acceso directo a localStorage en el entrypoint del framework.

#### 8. Escribir tests unitarios una vez aislada la lógica

- Tests unitarios para `domain` y `application` (use-cases) usando repositorios mock.
- Tests de integración para adaptadores (`LocalStorageCourseRepository`).

---

### Checklist rápido

- [ ] E2E cubriendo flujos críticos en el código legacy
- [ ] Interfaz `CourseRepository` y adaptación a localStorage
- [ ] Use-cases puros con inyección de dependencias
- [ ] Domain: value-objects y validadores
- [ ] `<source-root>/modules/<feature>/setup.[tj]s` creado como composition root por feature
- [ ] El entrypoint del framework reducido a entorno/arranque y consumo del `setup` del feature
- [ ] Migración completa a TypeScript y suite de tests unitarios funcionando

---

`<source-root>` es `src` si el proyecto ya usa `src/`; si no, es la raíz del repositorio.

**Resultado esperado:** Código desacoplado, testable y preparado para futuros cambios de infraestructura o framework sin tocar la lógica de negocio.
