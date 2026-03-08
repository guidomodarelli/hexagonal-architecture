# Named constructors en entidades y registro de eventos de dominio

Es el patrón que usamos en la entidad `Video` para registrar el evento de dominio `VideoCreatedDomainEvent` a la hora de crear nuevos videos.

```ts
abstract class AggregateRoot {
  private events: any[] = [];

  protected record(event: any): void {
    this.events.push(event);
  }
}

class VideoCreatedDomainEvent {
  constructor(
    public id: string,
    public details: {
      title: string,
      url: string,
      courseId: string
    }
  ) {}
}

class Video extends AggregateRoot {
  private constructor(
    private id: VideoId,
    private title: VideoTitle,
    private url: VideoURL,
    private courseId: CourseId
  ) {
    super();
  }

  static create(id: VideoId, title: VideoTitle, url: VideoURL, courseId: CourseId): Video {
    const video = new Video(id, title, url, courseId);

    video.record(new VideoCreatedDomainEvent(
      id.value(),
      {
        title: title.value(),
        url: url.value(),
        courseId: courseId.value()
      }
    ));

    return video;
  }
}
```

Es importante destacar que siempre registraremos los eventos de nuestras entidades en el punto exacto donde ocurran las acciones. Por ejemplo, el evento **VideoCreated** se ubica en el **NameConstructor** para garantizar que no se creen videos sin que se registre dicho evento. Del mismo modo, si se permitiera cambiar el título de los videos, existiría un método **Video#updateTitle** encargado de registrar el evento **VideoTitleUpdated**.

## 📢 Publicación de Eventos de Dominio

Una vez registrados los **eventos de dominio** generados durante la ejecución del caso de uso, el siguiente paso es **publicarlos** para que otras aplicaciones del sistema —o incluso otros módulos dentro de la misma aplicación— puedan aprovecharlos.

La publicación puede realizarse de diversas maneras. A continuación se detallan **tres enfoques posibles**, junto con sus ventajas y desventajas.

---

### 1️⃣ Uso de un *singleton* o método estático del publicador de eventos

**✅ Ventajas:**

* El evento se publica **en el punto más cercano a la modificación** del dominio.
  → Esto garantiza que no se pueda crear, por ejemplo, un *Video* sin que se publique el evento de dominio correspondiente.

**❌ Desventajas:**

* **Dependencias estáticas:**
  El uso de un método estático implica que las dependencias de `AwsSnsDomainEventPublisher` también deben ser estáticas, lo que introduce un acoplamiento rígido y problemático.
* **Violación de la regla de dependencia:**
  El modelo de dominio (*Video*) queda acoplado a una clase de infraestructura (`AwsSnsDomainEventPublisher`), lo cual **rompe el principio de inversión de dependencias** y el patrón de **puertos y adaptadores**.
* **Dificultad para testear:**
  La presencia de métodos estáticos complica la creación de *tests*. Aunque se podrían aplicar costuras para mitigar el problema, ello agrega complejidad innecesaria al código de producción.
* **Falta de margen de error:**
  Dado que el evento se publica antes de confirmar la persistencia del *Video*, un fallo en la base de datos podría dejar al sistema en un estado inconsistente: otros módulos asumirían que el evento es válido, aunque el registro nunca se haya guardado.

---

### 2️⃣ Inyección de un servicio de publicación de eventos en la entidad

**✅ Ventajas:**

* Se elimina el acoplamiento con la infraestructura.
* Se continúa publicando el evento **en el punto exacto donde ocurre la modificación**, manteniendo coherencia y proximidad con el cambio de estado.

**❌ Desventajas:**

* **Persisten los riesgos de inconsistencia:**
  El evento se publica antes de confirmar la persistencia, por lo que sigue sin haber margen de error.
* **Complejidad en la instanciación:**
  Cada vez que se cree una instancia del modelo de dominio (*Video*), será necesario pasarle una implementación de `DomainEventPublisher`. Esto afecta tanto al código de producción como a las pruebas automatizadas, aumentando la fricción en el desarrollo.

---

### 3️⃣ Inyección del publicador de eventos en el caso de uso

**✅ Ventajas:**

* **Consistencia arquitectónica:**
  Se sigue el mismo patrón que con otras dependencias (por ejemplo, el repositorio de persistencia).
  El publicador se inyecta en el **caso de uso**, aplicando el **principio de inversión de dependencias**.
  Cuantas menos excepciones haya en la forma de manejar dependencias, **más predecible y mantenible** será el código.
* **Mayor testeabilidad:**
  La clase de dominio (*Video*) no depende del publicador, lo que simplifica la comparación de instancias y la sustitución por *mocks* en pruebas.
* **Control del flujo de errores:**
  Si ocurre una excepción al guardar el *Video* en el repositorio, el flujo se detiene antes de publicar el evento.
  → De esta forma, ningún sistema o módulo externo asume la existencia de un *Video* que realmente no fue persistido.

**❌ Desventajas:**

* **Riesgo de omisión manual:**
  A diferencia de las alternativas anteriores, el evento **ya no se publica automáticamente** en el momento de la modificación.
  Si el *application service* olvida llamar al publicador, el evento quedará registrado pero no emitido, pasando inadvertido para el resto del sistema.

---

## 🧭 Conclusión

En nuestra opinión, **la tercera opción es la más adecuada** entre las tres alternativas analizadas.
Aunque presenta una posible desventaja, **consideramos que es perfectamente asumible** por los siguientes motivos:

### 🔍 Estrategias para mitigar el inconveniente de la opción 3️⃣

1️⃣ **Tests unitarios**

* Implementar pruebas unitarias que verifiquen la invocación al **`DomainEventPublisher`** durante la ejecución del caso de uso.
* Esto asegura que los eventos sean correctamente publicados y detecta de inmediato posibles omisiones.

2️⃣ **Flujo de trabajo colaborativo**

* Integrar un proceso de trabajo en equipo basado en ***pull requests***.
* Complementar este flujo con **revisiones de código** realizadas por los compañeros, garantizando así una segunda validación del cumplimiento de las buenas prácticas.

---

De este modo, incluso si se olvida publicar los eventos, estas dos medidas permiten **prevenir errores y mantener la calidad del código**.

## ¡Repasemos!

1. **¿Desde donde registraremos los eventos de dominio?**
  ✅ Desde la entidad y método donde se produzcan.

2. **¿Desde donde publicaremos los eventos de dominio?**
  ✅ Desde el caso de uso o ApplicationService ya que es éste quien representa la barrera a nivel de transacciones y publicación de eventos.