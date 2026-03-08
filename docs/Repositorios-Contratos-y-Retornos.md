# Repositorios: contratos y retornos

**Objetivo**: decidir qué retorna un repositorio y cuándo usar entidades de dominio, sin introducir DTOs de infraestructura fuera de su capa.

## Principios

- El puerto de repositorio (interface) modela necesidades del caso de uso, no detalles de red/DB.
- La implementación concreta (adaptador) mapea DTO externo ↔ modelo interno antes de cruzar el puerto.
- Priorizar siempre entidades de dominio como retorno.

```
Infra (DTO externo) → [mapper] → Dominio (Entidad)
```

## Contratos recomendados

### Escritura

- `create`: `Promise<User>` o `Promise<UserId>`
- `update`: `Promise<User>` o `Promise<void>`
- `delete`: `Promise<void>`

Razón: afectan invariantes del dominio; devolver la entidad/ID asegura consistencia y claridad del flujo. Nunca devuelvas un DTO externo.

### Lectura

- `findById`: `Promise<User | null>` (entidad de dominio)
- `search/findAll`: `Promise<User[]>` (entidad de dominio)

Razón: mantener consistencia usando siempre entidades de dominio simplifica el modelo mental y evita duplicación de estructuras.

## Dónde NO usar DTOs

- Los DTOs de infraestructura (HTTP/SDK/storage) se definen y usan solo en `infrastructure/api/dto` y vecinos; no aparecen en `application` ni `domain`.

## Ejemplo de puerto

```ts
// domain/repositories/UserRepository.ts
import { User } from '../../domain/User';

export interface UserRepository {
  create(name: string, email: string): Promise<User>;      // o Promise<UserId>
  findById(id: string): Promise<User | null>;
  findAll(): Promise<User[]>;
  update(user: User): Promise<User>;                       // o Promise<void>
}
```

## Adaptadores (infraestructura)

El adaptador importa entidades de dominio, nunca al revés. Mapea DTO externo → entidad.

```ts
// infrastructure/repositories/UserRepositoryFetch.ts
import { UserRepository } from '../../domain/repositories/UserRepository';
import { User } from '../../domain/User';
import { Email } from '../../domain/Email';
import { getUsers } from '../api/getUsers';

export class UserRepositoryFetch implements UserRepository {
  async findAll(): Promise<User[]> {
    const dtos = await getUsers();
    return dtos.map(dto => new User(dto.id, dto.name, new Email(dto.email)));
  }

  // ... resto de métodos
}
```

## Decisión rápida

- Retorna siempre entidades de dominio desde los puertos.
- Jamás retornes DTOs de infraestructura desde el puerto.

## Relación con los docs existentes

- DTOs de aplicación vs infraestructura: `docs/frontend-hexagonal/DTOs-Aplicacion-vs-Infraestructura.md`
- Guía general de DTOs/puertos/adaptadores: `docs/frontend-hexagonal/DTOs-Ports-Adapters.md`
- Ejemplos: `docs/frontend-hexagonal/examples/CreateUser.md` y `docs/frontend-hexagonal/examples/GetUsers.md`.
