# Repositorios: contratos y retornos

**Objetivo**: decidir qué retorna un puerto de escritura o lectura sin introducir DTOs de infraestructura fuera de su capa.

## Principios

- El puerto de repositorio (interface) modela necesidades del caso de uso, no detalles de red/DB.
- La implementación concreta (adaptador) mapea DTO externo ↔ modelo interno antes de cruzar el puerto.
- Para escrituras y flujos con invariantes, priorizar entidades de dominio o identificadores.
- Para lecturas, devolver entidades o read models internos según lo que realmente necesite el caso de uso.

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

- `findById`: `Promise<User | null>` cuando el caso de uso necesita una entidad de dominio
- `search/findAll`: `Promise<User[]>` o `Promise<UserListItem[]>`
- `dashboard/stats`: `Promise<UserMetricsResult>`

Razón: una lectura no siempre necesita comportamiento ni validaciones de una entidad completa. Si la operación solo proyecta datos para UI, reporting o búsqueda, un read model interno suele ser más claro y más estable que reutilizar una entidad por costumbre.

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

## Si la lectura necesita una proyección específica

Cuando una lectura no necesita una entidad rica, podés separar un puerto de lectura y devolver un resultado de aplicación:

```ts
// application/results/UserListItem.ts
export interface UserListItem {
  id: string;
  name: string;
  email: string;
}
```

```ts
// application/ports/UserListReader.ts
import { UserListItem } from '../results/UserListItem';

export interface UserListReader {
  list(): Promise<UserListItem[]>;
}
```

La regla sigue siendo la misma: el adaptador puede mapear desde DTOs externos, pero el núcleo solo ve modelos propios de `domain` o `application`.

## Adaptadores (infraestructura)

El adaptador importa modelos internos (`domain` o `application`), nunca al revés. Mapea DTO externo → entidad o DTO interno.

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

- En comandos y escrituras, priorizá entidades o IDs del dominio.
- En lecturas, usá entidades solo si el caso de uso necesita invariantes o comportamiento; si no, usá un read model interno.
- Jamás retornes DTOs de infraestructura desde el puerto.

## Relación con los docs existentes

- DTOs de aplicación vs infraestructura: `docs/frontend-hexagonal/DTOs-Aplicacion-vs-Infraestructura.md`
- Guía general de DTOs/puertos/adaptadores: `docs/frontend-hexagonal/DTOs-Ports-Adapters.md`
- Ejemplos: `docs/frontend-hexagonal/examples/CreateUser.md` y `docs/frontend-hexagonal/examples/GetUsers.md`.
