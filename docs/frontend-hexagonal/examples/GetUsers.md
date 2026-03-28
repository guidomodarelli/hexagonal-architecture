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

### Feature setup in `modules/users/setup.ts`

This example assumes the alias `@/` resolves to `<source-root>`.

```ts
import { GetUsers } from '@/modules/users/application/use-cases/GetUsers';
import { createBrowserHttpClient } from '@/modules/users/infrastructure/http/BrowserHttp/createBrowserHttpClient';
import { UserRepositoryHttp } from '@/modules/users/infrastructure/http/BrowserHttp/repositories/UserRepositoryHttp';

export function makeGetUsersHandler() {
  const httpClient = createBrowserHttpClient('/api');
  const userRepository = new UserRepositoryHttp(httpClient);

  return GetUsers({ userRepository });
}
```

### Usage from outer route / framework entrypoint

This file lives outside `modules/*` (for example in `app/routes/` or `src/pages/`). It is framework-shell code, not `infrastructure`.

```tsx
import { makeGetUsersHandler } from '@/modules/users/setup';
import { UsersSearchView } from '@/modules/users/presentation/pages/UsersSearchView';

const getUsers = makeGetUsersHandler();

export function UsersSearchRoute() {
  return <UsersSearchView onSearch={getUsers} />;
}
```

## Key Points

- `UserFilterInput` is an application DTO: internal contract between Presentation ↔ Application.
- `GetUsersResponseDto` and `UserDto` are infrastructure DTOs: external HTTP contract.
- The repository (adapter) translates external DTO → domain before returning the result to the use case.
- The use case keeps stable dependencies in the first call and runtime input in the second call.
- The adapter can stay class-based in `infrastructure` without changing the functional shape of the use case.
- An outer route or framework entrypoint imports from `modules/users/setup`; UI receives the composed handler by injection.
