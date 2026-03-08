# Organización de la infraestructura por implementación concreta

La capa de infraestructura aloja adaptadores para HTTP, persistencia, message brokers, email providers y cualquier otro mecanismo externo que implemente los puertos definidos en `domain/repositories`. Cada puerto puede tener múltiples implementaciones concretas (Axios, Fetch, PostgreSQL, MySQL, RabbitMQ, Kafka, Gmail, etc.), por lo que conviene aislar cada tecnología en su propia carpeta para facilitar el reemplazo sin tocar aplicación ni dominio.

## Objetivos

- Encapsular configuraciones, clientes y repositorios concretos por tecnología (`Axios`, `PostgreSQL`, `Kafka`, …).
- Compartir DTOs y mapeos desde `infrastructure/api/dto` para evitar duplicaciones.
- Hacer explícita la relación *puerto → adaptador* sin mezclar dependencias entre tecnologías.

## Estructura recomendada

```text
src/modules/<feature>/infrastructure/
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

Define cada puerto en la capa `domain` (`src/modules/<feature>/domain/services` o `src/modules/<feature>/domain/repositories`) y deja que la capa `infrastructure` solo implemente esas interfaces. Algunos ejemplos:

```ts
// src/modules/users/domain/services/HttpClient.ts
export interface HttpClient {
  get<T>(url: string): Promise<T>;
  post<T>(url: string, body: unknown): Promise<T>;
}
```

```ts
// src/modules/users/domain/services/DatabaseClient.ts
export interface DatabaseClient {
  query<T>(sql: string, params?: unknown[]): Promise<T[]>;
  transaction<T>(callback: () => Promise<T>): Promise<T>;
}
```

```ts
// src/modules/users/domain/services/MessageBrokerClient.ts
export interface MessageBrokerClient {
  publish(topic: string, payload: unknown): Promise<void>;
  subscribe(topic: string, handler: (payload: unknown) => Promise<void>): Promise<void>;
}
```

Cada implementación concreta se aloja en su carpeta correspondiente:

```ts
// src/modules/users/infrastructure/http/Axios/client/AxiosHttpClient.ts
import axios from 'axios';
import { HttpClient } from '../../../../domain/services/HttpClient';

export class AxiosHttpClient implements HttpClient {
  constructor(private readonly baseUrl: string) {}

  async get<T>(path: string): Promise<T> {
    const { data } = await axios.get<T>(`${this.baseUrl}${path}`);
    return data;
  }

  async post<T>(path: string, body: unknown): Promise<T> {
    const { data } = await axios.post<T>(`${this.baseUrl}${path}`, body);
    return data;
  }
}
```

```ts
// src/modules/users/infrastructure/persistence/PostgreSQL/repositories/PostgreSQLUserRepository.ts
import { UserRepository } from '../../../../domain/repositories/UserRepository';
import { DatabaseClient } from '../../../../domain/services/DatabaseClient';
import { UserDto } from '../../../api/dto/UserDto';
import { dtoToUser } from '../../../api/dto/mapper';

export class PostgreSQLUserRepository implements UserRepository {
  constructor(private readonly db: DatabaseClient) {}

  async findAll() {
    const rows = await this.db.query<UserDto>('SELECT * FROM users');
    return rows.map(dtoToUser);
  }
}
```

```ts
// src/modules/users/infrastructure/message-broker/Kafka/publishers/KafkaUserEventsPublisher.ts
import { MessageBrokerClient } from '../../../../domain/services/MessageBrokerClient';

export class KafkaUserEventsPublisher {
  constructor(private readonly client: MessageBrokerClient) {}

  async userCreated(payload: UserCreatedEvent) {
    await this.client.publish('users.created', payload);
  }
}
```

Cambiar de tecnología solo implica inyectar otra implementación (`FetchHttpClient`, `MySQLUserRepository`, `RabbitMQUserEventsPublisher`, etc.) manteniendo intacta la interfaz.

## DTOs siempre centralizados

La clave acá no es **entrada vs salida**, sino **adentro vs afuera** del hexágono:

- DTO interno (`application`/`domain`) → estable, definido por tus casos de uso y reglas de negocio.
- DTO externo (`infrastructure`) → variable, definido por APIs, bases de datos, message brokers, etc.

En esta organización:

- Los **DTO externos** (requests/responses HTTP, filas de base de datos, payloads de message brokers, SDKs de terceros) viven en `src/modules/<feature>/infrastructure/api/dto` y se traducen mediante un `mapper.ts`. Estas definiciones representan la forma con la que los servicios externos hablan con nosotros y no deben duplicarse en cada carpeta tecnológica (`Axios`, `Fetch`, `PostgreSQL`, etc.).
- Los **DTO internos** de la aplicación (por ejemplo, `CreateUserInput`, `GetUsersQuery`, `UserListResult`) viven en `src/modules/<feature>/application` (`commands`, `queries`, `results`) y son los que querés usar dentro del hexágono de punta a punta, independientemente de si afuera usás REST, gRPC, PostgreSQL, Mongo, etc.

Los repositorios y adaptadores de la capa `infrastructure` toman los DTO externos desde `infrastructure/api/dto`, los convierten a entidades de `domain` o a DTO internos, y solo exponen al resto de la aplicación modelos propios del núcleo (`domain` + `application`), nunca formatos crudos de `infrastructure`.
