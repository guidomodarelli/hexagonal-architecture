# Mejora los tests con Object Mother Pattern

## 🧩 Patrón **Object Mother**

El patrón **Object Mother** es una técnica que **simplifica y acelera** la creación de datos falsos en pruebas automatizadas, evitando la necesidad de instanciar manualmente objetos o definir valores fijos en cada test.

---

### ⚙️ Funcionamiento

En lugar de escribir los datos directamente dentro de los tests, se utilizan **funciones generadoras** que producen información dinámica y variada.
Para ello, se pueden emplear librerías como:

* **Faker** 🃏: genera valores aleatorios (nombres, direcciones, fechas, etc.).
* **Fishery** 🐟: permite construir funciones de tipo *Factory* para crear objetos de prueba de manera estructurada.

---

### 🚀 Beneficios principales

1️⃣ **Rapidez y eficiencia**
Facilita la preparación de datos de prueba, reduciendo el tiempo necesario para configurarlos.

2️⃣ **Robustez y fiabilidad**
La aleatoriedad en los datos evita probar siempre con los mismos valores, ayudando a descubrir errores ocultos.

3️⃣ **Legibilidad y mantenimiento**
Al encapsular los conjuntos de datos en objetos bien definidos, el código de los tests se vuelve más claro, conciso y fácil de mantener.


```typescript
import { CreateCourseForm } from './CreateCourseForm';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
// https://www.npmjs.com/package/@faker-js/faker
import { faker } from '@faker-js/faker';
// https://www.npmjs.com/package/fishery
import { Factory } from 'fishery';

type CreateCourseInput = {
  title: string;
  duration: number;
};

// Definimos una fábrica para crear el input del formulario
const createCourseInputFactory = Factory.define<CreateCourseInput>(() => ({
  title: faker.lorem.words(3), // Título aleatorio
  duration: faker.number.int({ min: 30, max: 180 }), // Duración aleatoria entre 30 y 180
}));

const CreateCourseInputMother = {
  create: (overrides?: Partial<CreateCourseInput>) => createCourseInputFactory.build(overrides),
  createMany: (count: number, overrides?: Partial<CreateCourseInput>) => createCourseInputFactory.buildList(count, overrides),
};

// Ejemplo de uso en un test
test('envía datos válidos', async () => {
  const user = userEvent.setup();
  const onCreate = jest.fn().mockResolvedValue(undefined);
  const courseInput = CreateCourseInputMother.create();
  render(<CreateCourseForm onCreate={onCreate} />);
  await user.type(screen.getByLabelText(/title/i), courseInput.title);
  await user.type(screen.getByLabelText(/duration/i), courseInput.duration.toString());
  await user.click(screen.getByRole('button', { name: /create/i }));
  expect(onCreate).toHaveBeenCalledWith(courseInput);
});
```
