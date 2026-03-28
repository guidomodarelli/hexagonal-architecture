# GetUsers Query Example (Read Operation)

Read use case with application-level filtering and external DTOs in infrastructure.

## Folder Structure (`<source-root>/modules/users/`)

`<source-root>` is `src` when the project uses `src/`; otherwise it is the repository root.

```text
<source-root>/modules/users/
├── setup.ts
├── presentation/
│   └── pages/
│       └── UsersSearchView.tsx
├── domain/
│   ├── User.ts
│   ├── Email.ts
│   └── repositories/
│       └── UserRepository.ts
├── application/
│   ├── use-cases/
│   │   └── GetUsers.ts
│   ├── queries/
│   │   └── UserFilterInput.ts
│   └── ports/
│       └── HttpClient.ts
└── infrastructure/
  ├── api/
  │   ├── getUsers.ts
  │   └── dto/
  │       ├── UserDto.ts
  │       ├── GetUsersResponseDto.ts
  │       └── mapper.ts
  ├── http/
  │   └── BrowserHttp/
  │       ├── createBrowserHttpClient.ts
  │       └── repositories/
  │           └── UserRepositoryHttp.ts
```

## Code

### domain/User.ts (Reusable with CreateUser)

```ts
import { Email } from './Email';

export class User {
  constructor(
  public readonly id: string,
  public readonly name: string,
  public readonly email: Email,
  ) {}
}
```

### domain/repositories/UserRepository.ts (Extended Port)

```ts
import { User } from '../User';

export interface UserRepository {
  // already exists in CreateUser:
  create(name: string, email: string): Promise<User>;

  // simple search/pagination:
  search(params: { query?: string; page?: number; limit?: number }): Promise<{ items: User[]; total: number }>;
}
```

### application/queries/UserFilterInput.ts

```ts
export interface UserFilterInput {
  query?: string;
  page?: number;  // 1-based
  limit?: number; // items per page
}
```

### application/use-cases/GetUsers.ts

```ts
import { UserRepository } from '../../domain/repositories/UserRepository';
import { User } from '../../domain/User';
import { UserFilterInput } from '../queries/UserFilterInput';

type GetUsersDependencies = {
  userRepository: UserRepository;
};

export const GetUsers =
  ({ userRepository }: GetUsersDependencies) =>
  async (input: UserFilterInput): Promise<{ items: User[]; total: number }> => {
    const page = input.page ?? 1;
    const limit = input.limit ?? 20;
    const query = input.query?.trim() || undefined;
    return userRepository.search({ query, page, limit });
  };
```

### application/ports/HttpClient.ts

```ts
export type HttpRequestConfig = {
  headers?: Record<string, string>;
  query?: Record<string, string | number | boolean | undefined>;
  timeoutMs?: number;
};

export type HttpResponse<T> = {
  status: number;
  data: T;
};

export interface HttpClient {
  // Resolves with { status, data } for any HTTP response, including 4xx/5xx.
  // Rejects only for transport-level failures such as network loss, timeout, or abort.
  get<T>(url: string, config?: HttpRequestConfig): Promise<HttpResponse<T>>;
  post<T>(url: string, body?: unknown, config?: HttpRequestConfig): Promise<HttpResponse<T>>;
}
```

### infrastructure/api/dto/UserDto.ts

```ts
export interface UserDto {
  id: string;
  name: string;
  email: string;
}
```

### infrastructure/api/dto/GetUsersResponseDto.ts

```ts
import { UserDto } from './UserDto';

export interface GetUsersResponseDto {
  items: UserDto[];
  total: number;
}
```

### infrastructure/api/dto/mapper.ts

```ts
import { User } from '../../../domain/User';
import { Email } from '../../../domain/Email';
import { UserDto } from './UserDto';

export function dtoToUser(dto: UserDto): User {
  return new User(dto.id, dto.name, new Email(dto.email));
}
```

### infrastructure/api/getUsers.ts

```ts
import { HttpClient } from '../../application/ports/HttpClient';
import { GetUsersResponseDto } from './dto/GetUsersResponseDto';

export async function getUsers(
  http: HttpClient,
  params: { query?: string; page?: number; limit?: number }
): Promise<GetUsersResponseDto> {
  const response = await http.get<GetUsersResponseDto>('/users', {
    query: {
      q: params.query,
      page: params.page,
      limit: params.limit,
    },
  });

  if (response.status < 200 || response.status >= 300) {
    throw new Error(`Could not fetch users: HTTP ${response.status}`);
  }

  return response.data;
}
```

### infrastructure/http/BrowserHttp/repositories/UserRepositoryHttp.ts (Adapter)

```ts
import { UserRepository } from '../../../../domain/repositories/UserRepository';
import { User } from '../../../../domain/User';
import { HttpClient } from '../../../../application/ports/HttpClient';
import { dtoToUser } from '../../../api/dto/mapper';
import { getUsers } from '../../../api/getUsers';

export class UserRepositoryHttp implements UserRepository {
  constructor(private readonly http: HttpClient) {}

  async create(_name: string, _email: string): Promise<User> {
    throw new Error('create() is covered in the CreateUser example');
  }

  async search(params: { query?: string; page?: number; limit?: number }) {
    const resp = await getUsers(this.http, params);
    return { items: resp.items.map(dtoToUser), total: resp.total };
  }
}
```

### Fragmento del builder en `modules/users/setup.ts`

Este ejemplo asume que el alias `@/` resuelve a `<source-root>`.
Si el mismo feature también expone otros use-cases (por ejemplo `createUser`), mantenelos en el mismo `buildUsersModule(...)`: este snippet muestra solo la parte relevante a `getUsers`.

```ts
import { GetUsers } from '@/modules/users/application/use-cases/GetUsers';
import type { UserRepository } from '@/modules/users/domain/repositories/UserRepository';

type UsersModuleDependencies = {
  userRepository: UserRepository;
};

export function buildUsersModule({ userRepository }: UsersModuleDependencies) {
  return {
    useCases: {
      // Otros use-cases del feature pueden convivir acá.
      getUsers: GetUsers({ userRepository }),
    },
  };
}
```

### Composition root global opcional en `modules/setup.ts`

```ts
import { buildUsersModule } from '@/modules/users/setup';
import { createBrowserHttpClient } from '@/modules/users/infrastructure/http/BrowserHttp/createBrowserHttpClient';
import { UserRepositoryHttp } from '@/modules/users/infrastructure/http/BrowserHttp/repositories/UserRepositoryHttp';

export function createRequestModules() {
  const httpClient = createBrowserHttpClient('/api');

  return {
    users: buildUsersModule({
      userRepository: new UserRepositoryHttp(httpClient),
    }),
  };
}
```

Si la app usa code splitting por ruta, este agregador debería reservarse para boundaries que realmente compartan wiring o expongan factories lazy; una route individual normalmente compone solo el feature que necesita.

### Uso desde la ruta externa o entrypoint del framework

Este archivo vive fuera de `modules/*` (por ejemplo en `app/routes/` o `src/pages/`). Es código del shell del framework, no `infrastructure`.

```tsx
import { buildUsersModule } from '@/modules/users/setup';
import { createBrowserHttpClient } from '@/modules/users/infrastructure/http/BrowserHttp/createBrowserHttpClient';
import { UserRepositoryHttp } from '@/modules/users/infrastructure/http/BrowserHttp/repositories/UserRepositoryHttp';
import { UsersSearchView } from '@/modules/users/presentation/pages/UsersSearchView';

const httpClient = createBrowserHttpClient('/api');
const usersModule = buildUsersModule({
  userRepository: new UserRepositoryHttp(httpClient),
});
const getUsers = usersModule.useCases.getUsers;

export function UsersSearchRoute() {
  return <UsersSearchView onSearch={getUsers} />;
}
```

## Puntos clave

- `UserFilterInput` es un DTO de `application`: contrato interno entre Presentation ↔ Application.
- `GetUsersResponseDto` y `UserDto` son DTOs de `infrastructure`: contrato HTTP externo.
- El repositorio (adapter) traduce DTO externo → domain antes de devolver el resultado al caso de uso.
- El caso de uso mantiene dependencias estables en la primera invocación e input de runtime en la segunda.
- El adapter puede seguir siendo una clase en `infrastructure` sin cambiar la forma funcional del caso de uso.
- Una ruta externa o entrypoint del framework suele componer solo el feature necesario con `buildUsersModule(...)`; `createRequestModules()` en `modules/setup` queda para wiring compartido o factories lazy. La UI recibe el caso de uso por inyección y la composición ocurre fuera del render.
