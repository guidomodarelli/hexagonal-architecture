# Programación funcional en arquitectura hexagonal

En el contexto de **JavaScript/TypeScript en frontend**, este repo prefiere un núcleo más **funcional y pragmático**. Eso no significa prohibir las clases en todo el sistema: significa que `application` y los servicios puros de `domain` suelen quedar más claros como funciones, mientras que `infrastructure` puede usar clases sin problema.

## Convención recomendada

### 1. Casos de uso con `dependency injection` por currying

La forma preferida para un caso de uso en frontend es:

```typescript
type CreateCourseDependencies = {
  courseRepository: CourseRepository;
  generateId: () => string;
};

export const CreateCourse =
  ({ courseRepository, generateId }: CreateCourseDependencies) =>
  async (request: CreateCourseRequest): Promise<void> => {
    const course: Course = {
      ...request,
      id: generateId(),
    };

    ensureCourseIsValid(course);
    await courseRepository.save(course);
  };
```

La regla práctica es:

- La **primera función** recibe dependencias estables del módulo.
- La **segunda función** recibe el input del caso de uso en runtime.
- Si aparece una tercera función, debe separar una preocupación real, no agregar ceremonia.

Por ejemplo, un tercer nivel puede tener sentido si querés separar `AbortSignal` o una policy opcional:

```typescript
const SearchCourses =
  ({ courseRepository }: SearchCoursesDependencies) =>
  (filters: SearchCoursesFilters) =>
  async (signal?: AbortSignal): Promise<Course[]> => {
    return courseRepository.search(filters, signal);
  };
```

## 2. Validaciones como funciones puras

No hace falta un `new Course()` para validar reglas simples. Podemos definir funciones independientes y reutilizables, por ejemplo en `CourseId.ts`, `CourseTitle.ts` o `CourseDuration.ts`.

```typescript
ensureCourseIsValid(course);
```

Beneficios:

- Funciones **puras y testeables** de forma aislada
- **Reutilizables en la lógica de UI**, manteniendo consistencia entre capas
- Separación clara de responsabilidades

## 3. Puertos explícitos, implementación flexible

El punto importante no es “todo como función” sino mantener **dependencias explícitas** y contratos estables.

El puerto puede seguir representándose como una `interface`:

```typescript
export interface CourseRepository {
  save(course: Course): Promise<void>;
  findAll(): Promise<Course[]>;
}
```

Y el caso de uso puede recibir ese puerto dentro de un objeto de dependencias:

```typescript
type CreateCourseDependencies = {
  courseRepository: CourseRepository;
};
```

Eso mantiene el núcleo funcional sin obligarte a modelar toda la infraestructura como funciones sueltas.

## 4. Infraestructura: las clases están bien

En `infrastructure`, las clases suelen ser una buena opción para adapters concretos porque encapsulan mejor:

- Clientes técnicos concretos (HTTP, SDKs, `Pool`, `Producer`, etc.)
- Configuración técnica (`baseUrl`, headers, credenciales, timeouts)
- Estado técnico o wiring propio del adaptador

```typescript
type HttpRequestConfig = {
  headers?: Record<string, string>;
  timeoutMs?: number;
};

type HttpResponse<T> = {
  status: number;
  data: T;
};

class HttpError extends Error {}

class HttpServerError extends HttpError {
  constructor(readonly status: number) {
    super(`HTTP ${status}`);
  }
}

class CourseRepositorySaveError extends Error {
  constructor(readonly cause: unknown) {
    super('Failed to save course');
  }
}

interface HttpClient {
  // Resolves with { status, data } for any HTTP response, including 4xx/5xx.
  // Rejects only for transport-level failures such as network loss, timeout, or abort.
  post<TResponse>(
    url: string,
    body?: unknown,
    config?: HttpRequestConfig
  ): Promise<HttpResponse<TResponse>>;
}

export class CourseRepositoryHttp implements CourseRepository {
  constructor(private readonly http: HttpClient) {}

  async save(course: Course): Promise<void> {
    let response: HttpResponse<void>;

    try {
      response = await this.http.post<void>('/courses', course, {
        headers: {
          'Content-Type': 'application/json',
        },
      });
    } catch (error) {
      throw new CourseRepositorySaveError(error);
    }

    if (response.status >= 500) {
      throw new CourseRepositorySaveError(new HttpServerError(response.status));
    }

    if (response.status < 200 || response.status >= 300) {
      throw new CourseRepositorySaveError(new HttpError(`HTTP ${response.status}`));
    }
  }
}
```

Luego `modules/<feature>/setup.ts` arma el builder del módulo:

```typescript
import { CreateCourse } from '@/modules/courses/application/use-cases/CreateCourse';
import type { CourseRepository } from '@/modules/courses/domain/repositories/CourseRepository';

export function buildCoursesModule({
  courseRepository,
  generateId,
}: {
  courseRepository: CourseRepository;
  generateId: () => string;
}) {
  return {
    useCases: {
      createCourse: CreateCourse({
        courseRepository,
        generateId,
      }),
    },
  };
}
```

Y, si querés centralizar el wiring, `modules/setup.ts` puede actuar como agregador opcional:

```typescript
import { buildCoursesModule } from '@/modules/courses/setup';
import { createBrowserHttpClient } from '@/modules/courses/infrastructure/http/BrowserHttp/createBrowserHttpClient';
import { CourseRepositoryHttp } from '@/modules/courses/infrastructure/http/BrowserHttp/repositories/CourseRepositoryHttp';

export function createRequestModules() {
  const httpClient = createBrowserHttpClient('/api');

  return {
    courses: buildCoursesModule({
      courseRepository: new CourseRepositoryHttp(httpClient),
      generateId: () => crypto.randomUUID(),
    }),
  };
}
```

---

## Resumen

Este enfoque aprovecha la **naturaleza funcional de JavaScript** sin volver dogmática la arquitectura:

- ✅ **Validaciones**: funciones puras, reutilizables y testeables
- ✅ **Casos de uso**: DI por currying para separar configuración de ejecución
- ✅ **Infraestructura**: adapters concretos pueden ser clases si eso hace más claro el borde técnico

## ¿Tienen sentido los Value Objects?

En programación orientada a objetos, un **Value Object** encapsula un valor y concentra su lógica asociada, evitando que esta se disperse en la entidad principal. Por ejemplo, en lugar de que `Course` tenga propiedades primitivas (`string`, `number`), cada una se representa mediante su propio Value Object: `CourseTitle`, `ImageUrl`, `CourseId`, etc.

**Ventaja clave:** las validaciones (longitud, formato, rangos) viven en el Value Object correspondiente, no en la entidad.

---

### Aplicación en frontend: Value Files

En frontend podemos adoptar el mismo patrón de forma **funcional y ligera**. Para reglas simples, solemos usar **archivos independientes** (Value Files) que exportan:

1. **Tipo semántico** (alias sobre primitivos)
2. **Reglas de validación**
3. **Funciones auxiliares** (errores, normalizaciones)

**Ejemplo** (`CourseTitle.ts`):

```typescript
// Tipo semántico
export type CourseTitle = string;

// Constantes de validación
export const COURSE_TITLE_MIN_LENGTH = 5;
export const COURSE_TITLE_MAX_LENGTH = 100;

// Validación
export function isCourseTitleValid(title: string): boolean {
  return (
    title.length >= COURSE_TITLE_MIN_LENGTH &&
    title.length <= COURSE_TITLE_MAX_LENGTH
  );
}

// Error asociado
export function CourseTitleNotValidError(title: string): Error {
  return new Error(`Title "${title}" is not valid.`);
}
```

La interfaz `Course` usa tipos semánticos en lugar de primitivos:

```typescript
export interface Course {
  id: CourseId;
  title: CourseTitle;
  imageUrl: ImageUrl;
}
```

---

### Beneficios

- **Semántica rica:** `CourseTitle` expresa mejor el dominio que `string`
- **Consistencia:** las reglas viven junto al valor que gobiernan
- **Evolutivo:** empieza con un alias simple, añade validaciones cuando sea necesario
- **Testeable:** funciones puras, fáciles de probar en aislamiento
- **Sin overhead:** no requiere instancias ni constructores

---

**Conclusión:** Los Value Objects tienen pleno sentido en frontend mediante un enfoque pragmático: **tipos alias + funciones puras** en archivos separados cuando alcanza, o clases si el dominio realmente gana algo con ello. La preferencia de este repo es mantener funcionales los casos de uso y servicios puros del núcleo, y usar clases sobre todo en adapters de `infrastructure`.
