# Añade validaciones a la creación del curso

Tenemos nuestro caso de uso para crear cursos, pero necesitamos añadir validaciones para garantizar la integridad de los datos antes de persistirlos.

Implementaremos validaciones en la capa de **Dominio** siguiendo el principio de que las reglas de negocio deben residir en el dominio, no en la interfaz de usuario. Esto nos permite:

- Reutilizar las validaciones en diferentes contextos (formularios, APIs, tests)
- Mantener la lógica de negocio centralizada y consistente
- Validar tanto en el frontend (feedback inmediato) como en el backend

Por ejemplo, para el título del curso estableceremos restricciones de longitud mínima y máxima de caracteres.

`<source-root>` es `src` si el proyecto usa `src/`; si no, es la raíz del repositorio.

## Value Object: CourseTitle

Creamos un Value Object en la capa de **Dominio** para encapsular la lógica de validación del título del curso.

```typescript
// <source-root>/modules/courses/domain/value-objects/CourseTitle.ts
export class CourseTitleNotValidError extends Error {
  constructor(title: string) {
    super(`Course title not valid: ${title}`);
    this.name = 'CourseTitleNotValidError';
  }
}

export const MIN_COURSE_TITLE_LENGTH = 5;
export const MAX_COURSE_TITLE_LENGTH = 100;

export const isCourseTitleValid = (title: string): boolean => {
  return title.length >= MIN_COURSE_TITLE_LENGTH && title.length <= MAX_COURSE_TITLE_LENGTH;
};
```

## Validador de entidad: CourseValidator

Creamos un validador en la capa de **Dominio** que verifica todos los atributos del curso de forma centralizada. Esta función lanza errores específicos cuando detecta valores inválidos, permitiendo identificar exactamente qué campo no cumple los requisitos.

```typescript
// <source-root>/modules/courses/domain/entities/CourseValidator.ts
import { Course } from './Course';
import { isCourseIdValid, CourseIdNotValidError } from '../value-objects/CourseId';
import { isCourseTitleValid, CourseTitleNotValidError } from '../value-objects/CourseTitle';
import { isCourseDurationValid, CourseDurationNotValidError } from '../value-objects/CourseDuration';

export const ensureCourseIsValid = (course: Course): void => {
  if (!isCourseIdValid(course.id)) {
    throw new CourseIdNotValidError(course.id);
  }
  if (!isCourseTitleValid(course.title)) {
    throw new CourseTitleNotValidError(course.title);
  }
  if (!isCourseDurationValid(course.duration)) {
    throw new CourseDurationNotValidError(course.duration);
  }
  // Aquí podríamos añadir más validaciones para otros campos del curso
};
```

Esta función se invoca en el caso de uso `CreateCourse` antes de persistir el curso en el repositorio.

## Validación en tiempo real en el formulario

Para proporcionar feedback inmediato al usuario mientras completa los campos, utilizamos las funciones de validación individuales:

```typescript
import { useEffect, useState } from 'react';
import {
  isCourseTitleValid,
  MIN_COURSE_TITLE_LENGTH,
  MAX_COURSE_TITLE_LENGTH
} from '../../domain/value-objects/CourseTitle';
import { isCourseDurationValid } from '../../domain/value-objects/CourseDuration';

useEffect(() => {
  const isTitleValid = isCourseTitleValid(formData.title);
  const isDurationValid = isCourseDurationValid(formData.duration);

  setErrors({
    title: isTitleValid
      ? null
      : `El título debe tener entre ${MIN_COURSE_TITLE_LENGTH} y ${MAX_COURSE_TITLE_LENGTH} caracteres.`,
    duration: isDurationValid
      ? null
      : 'La duración debe ser un número positivo.',
  });
}, [formData]);
```

De este modo, el usuario recibe retroalimentación instantánea sobre la validez de los datos mientras escribe, sin necesidad de enviar el formulario. Esto mejora significativamente la experiencia de usuario.
