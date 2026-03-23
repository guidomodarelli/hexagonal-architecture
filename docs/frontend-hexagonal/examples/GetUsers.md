# GetUsers Query Example (Read Operation)

Read use case with application-level filtering and external DTOs in infrastructure.

## Folder Structure (`src/modules/users/`)

```text
src/modules/users/
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
└── infrastructure/
  ├── api/
  │   ├── getUsers.ts
  │   └── dto/
  │       ├── UserDto.ts
  │       ├── GetUsersResponseDto.ts
  │       └── mapper.ts
  └── repositories/
    └── UserRepositoryFetch.ts
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

export function GetUsers(
  repo: UserRepository
): (input: UserFilterInput) => Promise<{ items: User[]; total: number }> {
  return async (input: UserFilterInput): Promise<{ items: User[]; total: number }> => {
    const page = input.page ?? 1;
    const limit = input.limit ?? 20;
    const query = input.query?.trim() || undefined;
    return repo.search({ query, page, limit });
  };
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
import { GetUsersResponseDto } from './dto/GetUsersResponseDto';

export async function getUsers(params: { query?: string; page?: number; limit?: number }): Promise<GetUsersResponseDto> {
  const url = new URL('/api/users', window.location.origin);
  if (params.query) url.searchParams.set('q', params.query);
  if (params.page) url.searchParams.set('page', String(params.page));
  if (params.limit) url.searchParams.set('limit', String(params.limit));

  const res = await fetch(url.toString(), { method: 'GET' });
  if (!res.ok) throw new Error('Could not fetch users');
  return res.json();
}
```

### infrastructure/repositories/UserRepositoryFetch.ts (Adapter)

```ts
import { UserRepository } from '../../domain/repositories/UserRepository';
import { User } from '../../domain/User';
import { dtoToUser } from '../api/dto/mapper';
import { getUsers } from '../api/getUsers';

export class UserRepositoryFetch implements UserRepository {
  async create(_name: string, _email: string): Promise<User> {
    throw new Error('create() is covered in the CreateUser example');
  }

  async search(params: { query?: string; page?: number; limit?: number }) {
  const resp = await getUsers(params);
  return { items: resp.items.map(dtoToUser), total: resp.total };
  }
}
```

### Composition root / feature bootstrap

This example assumes the alias `@/` resolves to `src/`.

```ts
import { GetUsers } from '@/modules/users/application/use-cases/GetUsers';
import { UserRepositoryFetch } from '@/modules/users/infrastructure/repositories/UserRepositoryFetch';

export const getUsers = GetUsers(new UserRepositoryFetch());
```

### Usage from UI

```ts
import { getUsers } from '@/app/users/getUsers';

export async function searchUsersFromUI(query: string) {
  return getUsers({ query, page: 1, limit: 20 });
}
```

## Key Points

- `UserFilterInput` is an application DTO: internal contract between Presentation ↔ Application.
- `GetUsersResponseDto` and `UserDto` are infrastructure DTOs: external HTTP contract.
- The repository (adapter) translates external DTO → domain before returning the result to the use case.
- UI imports a composed use case or handler, not the infrastructure adapter directly.
