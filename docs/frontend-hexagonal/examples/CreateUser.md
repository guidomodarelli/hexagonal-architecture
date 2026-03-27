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
└── infrastructure/
  ├── api/
  │   ├── createUser.ts
  │   └── dto/
  │       ├── CreateUserDto.ts
  │       ├── UserDto.ts
  │       └── mapper.ts
  └── repositories/
    └── UserRepositoryFetch.ts
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
import { CreateUserDto } from './dto/CreateUserDto';
import { UserDto } from './dto/UserDto';

export async function createUser(dto: CreateUserDto): Promise<UserDto> {
  const res = await fetch('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(dto),
  });
  if (!res.ok) throw new Error('Could not create user');
  return res.json();
}
```

### infrastructure/repositories/UserRepositoryFetch.ts (Adapter)

```ts
import { UserRepository } from '../../domain/repositories/UserRepository';
import { User } from '../../domain/User';
import { createUser } from '../api/createUser';
import { dtoToUser } from '../api/dto/mapper';

export class UserRepositoryFetch implements UserRepository {
  async create(name: string, email: string): Promise<User> {
  const userDto = await createUser({ name, email });
  return dtoToUser(userDto);
  }
}
```

### Feature setup in `modules/users/setup.ts`

This example assumes the alias `@/` resolves to `<source-root>`.

```ts
import { CreateUser } from '@/modules/users/application/use-cases/CreateUser';
import { UserRepositoryFetch } from '@/modules/users/infrastructure/repositories/UserRepositoryFetch';

export function makeCreateUserHandler() {
  const userRepository = new UserRepositoryFetch();

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

Moving the `fetch` call directly into the repo is valid if you don't need to reuse or test it separately.

```ts
export class UserRepositoryFetch implements UserRepository {
  async create(name: string, email: string): Promise<User> {
  const res = await fetch('/api/users', { method: 'POST', body: JSON.stringify({ name, email }) });
  if (!res.ok) throw new Error('Could not create user');
  const dto = await res.json();
  return dtoToUser(dto);
  }
}
```

## Key points

- DTOs (external) in `infrastructure/api/dto`. Internal use case inputs in `application/commands`.
- Port in `domain/repositories`. Adapter in `infrastructure/repositories`.
- The use case keeps stable dependencies in the first call and runtime input in the second call.
- The adapter can stay class-based in `infrastructure` without changing the functional shape of the use case.
- An outer route or framework entrypoint imports from `modules/users/setup`; UI receives the composed handler by injection.
- infrastructure can import domain and application. Application and domain don't import infrastructure.
