# ¿Diferencia entre View y Component?

## 🧱 Arquitectura de la Interfaz

### 🖥️ **View (Page)**

La **View** —también llamada *Page*— se encarga de **orquestar toda la pantalla**.
Sus responsabilidades principales son:

1️⃣ **Composición:** agrupa y organiza los distintos componentes visuales que conforman la página.
2️⃣ **Gestión de dependencias:** consume **casos de uso** o **handlers** ya compuestos desde `main.ts`, un *feature bootstrap* o un contenedor de dependencias. Puede decidir qué dependencias de presentación pasar al componente, pero evita instanciar adaptadores concretos en la propia View.
3️⃣ **Manejo de efectos secundarios:** controla los **side-effects** externos (peticiones, listeners, suscripciones, etc.).
4️⃣ **Sin reglas de dominio:** no contiene lógica de negocio; su función es puramente estructural y de coordinación.

---

### 🧩 **Component**

El **Component** es una **pieza reutilizable de la UI** con su propia **lógica de presentación**.
Sus funciones principales son:

1️⃣ **Estado y eventos:** gestiona su estado interno, las interacciones del usuario y los eventos de interfaz.
2️⃣ **Validaciones de UI:** controla aspectos visuales y validaciones relacionadas con la experiencia del usuario.
3️⃣ **Sin lógica de negocio:** no implementa reglas de dominio; se comunica con los **casos de uso** mediante **props** o **hooks**, delegando la lógica compleja a otros niveles.


Ejemplo mínimo (React) con dependencias mínimas

Los imports con `@/` asumen un alias configurado hacia `src/`.
Para mantener el foco, el ejemplo usa solo `title` y `duration`; si tu caso real incluye más campos (por ejemplo `description`), la separación entre `View` y `Component` no cambia.

## Componente de formulario (solo React)

```tsx
import React, { useState } from 'react';
import { isCourseTitleValid } from '@/modules/courses/domain/value-objects/CourseTitle';
import { isCourseDurationValid } from '@/modules/courses/domain/value-objects/CourseDuration';

type CreateInput = { title: string; duration: number };

interface Props {
  onCreate: (data: CreateInput) => Promise<void>
};

export function CreateCourseForm({ onCreate }: Props) {
  const [title, setTitle] = useState('');
  const [duration, setDuration] = useState(''); // como texto para el input

  const isTitleValid = isCourseTitleValid(title);
  const isDurationValid = isCourseDurationValid(Number(duration));
  const canSubmit = isTitleValid && isDurationValid;

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!canSubmit) return;
    await onCreate({ title, duration: Number(duration) });
    setTitle('');
    setDuration('');
  };

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Title
        <input
          aria-label="title"
          value={title}
          onChange={(e) => setTitle(e.target.value)}
        />
      </label>

      <label>
        Duration
        <input
          aria-label="duration"
          value={duration}
          onChange={(e) => setDuration(e.target.value)}
        />
      </label>

      {!isTitleValid && <p role="alert">Title must be 5–100 chars</p>}
      {!isDurationValid && <p role="alert">Duration must be > 0</p>}

      <button type="submit" disabled={!canSubmit}>
        Create
      </button>
    </form>
  );
}
```

## Composition root mínimo (fuera de la View)

```ts
import { CreateCourse } from '@/modules/courses/application/use-cases/CreateCourse';
import { CourseRepositoryFetch } from '@/modules/courses/infrastructure/repositories/CourseRepositoryFetch';

const repo = new CourseRepositoryFetch();
export const createCourse = CreateCourse(repo);
```

## View que consume un caso de uso ya compuesto

```tsx
import React from 'react';
import { CreateCourseForm } from './CreateCourseForm';
import { createCourse } from '@/app/courses/createCourse';

export function CreateCourseView() {
  return <CreateCourseForm onCreate={createCourse} />;
}
```

Si preferís usar un contenedor de dependencias o una factory por módulo, el principio no cambia: la View **consume** la dependencia compuesta, no crea adaptadores concretos.
