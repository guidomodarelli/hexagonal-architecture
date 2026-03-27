# Organización de la infraestructura por implementación concreta

La capa de infraestructura aloja adaptadores para HTTP, persistencia, message brokers, email providers y cualquier otro mecanismo externo que implemente puertos definidos en `domain/repositories` y, cuando haga falta, contratos internos en `application/ports`. Cada puerto puede tener múltiples implementaciones concretas (Axios, Fetch, PostgreSQL, MySQL, RabbitMQ, Kafka, Gmail, etc.), por lo que conviene aislar cada tecnología en su propia carpeta para facilitar el reemplazo sin tocar aplicación ni dominio.

## Objetivos

- Encapsular configuraciones, clientes y repositorios concretos por tecnología (`Axios`, `PostgreSQL`, `Kafka`, …).
- Compartir DTOs y mapeos desde `infrastructure/api/dto` para evitar duplicaciones.
- Hacer explícita la relación *puerto → adaptador* sin mezclar dependencias entre tecnologías.

## Estructura recomendada

`<source-root>` es `src` si el proyecto usa `src/`; si no, es la raíz del repositorio.

```text
<source-root>/modules/<feature>/infrastructure/
  http/
    Axios/
      client/
      repositories/
      tests/
    Fetch/
      client/
      repositories/
  persistence/
    PostgreSQL/
      migrations/
      repositories/
    MySQL/
      repositories/
  message-broker/
    RabbitMQ/
      client/
      publishers/
    Kafka/
      client/
      publishers/
  email/
    Gmail/
      adapters/
    Resend/
      adapters/
  api/
    dto/
      ...
```

- Cada carpeta de primer nivel (`http`, `persistence`, `message-broker`, `email`, etc.) agrupa implementaciones de un tipo de infraestructura.
- Dentro de cada tipo, crea carpetas por tecnología concreta (`Axios`, `PostgreSQL`, `Kafka`, …) que contengan clientes, helpers y repositorios específicos.
- Los DTOs externos y sus mapeos viven en `infrastructure/api/dto` sin importar la tecnología consumida.

## Contratos y adaptadores

No todo contrato técnico debe subir al dominio.

- Los **puertos de negocio** viven en `domain/repositories` o `domain/services` cuando expresan lenguaje del negocio.
- Los **contratos internos de aplicación** viven en `application/ports` cuando coordinan un caso de uso pero no forman parte del lenguaje ubicuo.
- Los **clientes técnicos de bajo nivel** (`axios.create()`, pools de base de datos, SDK wrappers genéricos) pueden quedarse como detalle privado de `infrastructure`.

Ejemplos:

```ts
// <source-root>/modules/users/domain/repositories/UserRepository.ts
export interface UserRepository {
  findAll(): Promise<User[]>;
}
```

```ts
// <source-root>/modules/users/application/ports/UserEventsPublisher.ts
export interface UserEventsPublisher {
  userCreated(payload: UserCreatedEvent): Promise<void>;
}
```

```ts
// <source-root>/modules/users/infrastructure/http/Axios/client/createAxiosClient.ts
import axios, { AxiosInstance } from 'axios';

export function createAxiosClient(baseUrl: string): AxiosInstance {
  return axios.create({ baseURL: baseUrl });
}
```

Cada implementación concreta se aloja en su carpeta correspondiente:

```ts
// <source-root>/modules/users/infrastructure/persistence/PostgreSQL/repositories/PostgreSQLUserRepository.ts
import { Pool } from 'pg';
import { UserRepository } from '../../../../domain/repositories/UserRepository';
import { UserDto } from '../../../api/dto/UserDto';
import { dtoToUser } from '../../../api/dto/mapper';

export class PostgreSQLUserRepository implements UserRepository {
  constructor(private readonly pool: Pool) {}

  async findAll() {
    const result = await this.pool.query<UserDto>('SELECT id, name, email FROM users');
    return result.rows.map(dtoToUser);
  }
}
```

```ts
// <source-root>/modules/users/infrastructure/message-broker/Kafka/publishers/KafkaUserEventsPublisher.ts
import type { Producer } from 'kafkajs';
import { UserEventsPublisher } from '../../../../application/ports/UserEventsPublisher';

export class KafkaUserEventsPublisher implements UserEventsPublisher {
  constructor(private readonly producer: Producer) {}

  async userCreated(payload: UserCreatedEvent) {
    await this.producer.send({
      topic: 'users.created',
      messages: [{ value: JSON.stringify(payload) }],
    });
  }
}
```

Cambiar de tecnología solo implica inyectar otra implementación (`Fetch`, `MySQL`, `RabbitMQ`, etc.) manteniendo intacto el contrato expuesto al núcleo.

## DTOs siempre centralizados

La clave acá no es **entrada vs salida**, sino **adentro vs afuera** del hexágono:

- DTO interno (`application`/`domain`) → estable, definido por tus casos de uso y reglas de negocio.
- DTO externo (`infrastructure`) → variable, definido por APIs, bases de datos, message brokers, etc.

En esta organización:

- Los **DTO externos** (requests/responses HTTP, filas de base de datos, payloads de message brokers, SDKs de terceros) viven en `<source-root>/modules/<feature>/infrastructure/api/dto` y se traducen mediante un `mapper.ts`. Estas definiciones representan la forma con la que los servicios externos hablan con nosotros y no deben duplicarse en cada carpeta tecnológica (`Axios`, `Fetch`, `PostgreSQL`, etc.).
- Los **DTO internos** de la aplicación (por ejemplo, `CreateUserInput`, `GetUsersQuery`, `UserListResult`) viven en `<source-root>/modules/<feature>/application` (`commands`, `queries`, `results`) y son los que querés usar dentro del hexágono de punta a punta, independientemente de si afuera usás REST, gRPC, PostgreSQL, Mongo, etc.

Los repositorios y adaptadores de la capa `infrastructure` toman los DTO externos desde `infrastructure/api/dto`, los convierten a entidades de `domain` o a DTO internos, y solo exponen al resto de la aplicación modelos propios del núcleo (`domain` + `application`), nunca formatos crudos de `infrastructure`.
