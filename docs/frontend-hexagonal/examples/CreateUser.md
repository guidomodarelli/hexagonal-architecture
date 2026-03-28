# Complete Example: CreateUser

Realistic example with modules, ports, adapters (repository), DTOs and mappers.

## Folder tree (`<source-root>/modules/users/`)

`<source-root>` is `src` when the project uses `src/`; otherwise it is the repository root.

```text
<source-root>/modules/users/
├── setup.ts
├── presentation/
│   └── pages/
│       └── CreateUserView.tsx
├── domain/
│   ├── User.ts
│   ├── Email.ts
│   └── repositories/
│       └── UserRepository.ts
├── application/
│   ├── use-cases/
│   │   └── CreateUser.ts
│   ├── commands/
│   │   └── CreateUserInput.ts
│   └── ports/
│       └── HttpClient.ts
└── infrastructure/
  ├── api/
  │   ├── createUser.ts
  │   └── dto/
  │       ├── CreateUserDto.ts
  │       ├── UserDto.ts
  │       └── mapper.ts
  ├── http/
  │   └── BrowserHttp/
  │       ├── createBrowserHttpClient.ts
  │       └── repositories/
  │           └── UserRepositoryHttp.ts
```

## Code

### domain/User.ts

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

### domain/Email.ts

```ts
export class Email {
  constructor(private readonly rawValue: string) {
  if (!rawValue.includes('@')) throw new Error('Invalid email');
  }
  get value(): string {
  return this.rawValue;
  }
}
```

### domain/repositories/UserRepository.ts (Port)

```ts
import { User } from '../User';

export interface UserRepository {
  create(name: string, email: string): Promise<User>;
}
```

### application/commands/CreateUserInput.ts

```ts
export interface CreateUserInput {
  name: string;
  email: string;
}
```

### application/use-cases/CreateUser.ts

```ts
import { UserRepository } from '../../domain/repositories/UserRepository';
import { User } from '../../domain/User';
import { Email } from '../../domain/Email';
import { CreateUserInput } from '../commands/CreateUserInput';

type CreateUserDependencies = {
  userRepository: UserRepository;
};

export const CreateUser =
  ({ userRepository }: CreateUserDependencies) =>
  async (input: CreateUserInput): Promise<User> => {
    const emailVO = new Email(input.email);
    return userRepository.create(input.name, emailVO.value);
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

### infrastructure/api/dto/CreateUserDto.ts

```ts
export interface CreateUserDto {
  name: string;
  email: string;
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

### infrastructure/api/dto/mapper.ts

```ts
import { UserDto } from './UserDto';
import { User } from '../../../domain/User';
import { Email } from '../../../domain/Email';

export function dtoToUser(dto: UserDto): User {
  return new User(dto.id, dto.name, new Email(dto.email));
}
```

### infrastructure/api/createUser.ts

```ts
import { HttpClient } from '../../application/ports/HttpClient';
import { CreateUserDto } from './dto/CreateUserDto';
import { UserDto } from './dto/UserDto';

export async function createUser(
  http: HttpClient,
  dto: CreateUserDto
): Promise<UserDto> {
  const response = await http.post<UserDto>('/users', dto, {
    headers: { 'Content-Type': 'application/json' },
  });

  if (response.status < 200 || response.status >= 300) {
    throw new Error(`Could not create user: HTTP ${response.status}`);
  }

  return response.data;
}
```

### infrastructure/http/BrowserHttp/repositories/UserRepositoryHttp.ts (Adapter)

```ts
import { UserRepository } from '../../../../domain/repositories/UserRepository';
import { User } from '../../../../domain/User';
import { HttpClient } from '../../../../application/ports/HttpClient';
import { createUser } from '../../../api/createUser';
import { dtoToUser } from '../../../api/dto/mapper';

export class UserRepositoryHttp implements UserRepository {
  constructor(private readonly http: HttpClient) {}

  async create(name: string, email: string): Promise<User> {
    const userDto = await createUser(this.http, { name, email });
    return dtoToUser(userDto);
  }
}
```

### Feature setup in `modules/users/setup.ts`

This example assumes the alias `@/` resolves to `<source-root>`.

```ts
import { CreateUser } from '@/modules/users/application/use-cases/CreateUser';
import { createBrowserHttpClient } from '@/modules/users/infrastructure/http/BrowserHttp/createBrowserHttpClient';
import { UserRepositoryHttp } from '@/modules/users/infrastructure/http/BrowserHttp/repositories/UserRepositoryHttp';

export function makeCreateUserHandler() {
  const httpClient = createBrowserHttpClient('/api');
  const userRepository = new UserRepositoryHttp(httpClient);

  return CreateUser({ userRepository });
}
```

### Usage from outer route / framework entrypoint

This file lives outside `modules/*` (for example in `app/routes/` or `src/pages/`). It is framework-shell code, not `infrastructure`.

```tsx
import { makeCreateUserHandler } from '@/modules/users/setup';
import { CreateUserView } from '@/modules/users/presentation/pages/CreateUserView';

const createUser = makeCreateUserHandler();

export function CreateUserRoute() {
  return <CreateUserView onCreate={createUser} />;
}
```

## Minimalist variant (without separate `api/`)

Moving the HTTP call directly into the repo is valid if you don't need to reuse or test it separately.

```ts
export class UserRepositoryHttp implements UserRepository {
  constructor(private readonly http: HttpClient) {}

  async create(name: string, email: string): Promise<User> {
    const response = await this.http.post<UserDto>(
      '/users',
      { name, email },
      { headers: { 'Content-Type': 'application/json' } }
    );

    if (response.status < 200 || response.status >= 300) {
      throw new Error(`Could not create user: HTTP ${response.status}`);
    }

    return dtoToUser(response.data);
  }
}
```

## Key points

- DTOs (external) in `infrastructure/api/dto`. Internal use case inputs in `application/commands`.
- Port in `domain/repositories`. Adapter in `infrastructure/http/BrowserHttp/repositories` (or another concrete implementation folder).
- The use case keeps stable dependencies in the first call and runtime input in the second call.
- The adapter can stay class-based in `infrastructure` without changing the functional shape of the use case.
- An outer route or framework entrypoint imports from `modules/users/setup`; UI receives the composed handler by injection.
- infrastructure can import domain and application. Application and domain don't import infrastructure.
