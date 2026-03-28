# DTOs en Aplicación vs Infraestructura: cuándo, desde dónde y por qué

Esta guía complementa `DTOs-Ports-Adapters.md` diferenciando explícitamente los DTOs de **aplicación** (inputs/outputs internos de casos de uso) y los DTOs de **infraestructura** (contratos con el exterior).

> Terminología: usamos "DTO interno" o "DTO de aplicación" para inputs/outputs de casos de uso dentro del hexágono; y "DTO externo" o "DTO de infraestructura" para requests/responses de HTTP/SDK/storage, filas de base de datos, SDKs de terceros, etc.

La clave no es tanto **entrada vs salida**, sino **adentro vs afuera** de tu hexágono:

- DTO interno (aplicación/dominio) → contrato estable definido por tu negocio.
- DTO externo (infraestructura) → contrato variable definido por APIs/DB/clientes.

Idealmente, un cambio en la infraestructura (pasar de REST a gRPC, de un cliente HTTP a otro, de PostgreSQL a Mongo, de una API externa a otra) no debería obligarte a cambiar tus DTO internos, solo los externos y sus mapeos.

---

## DTOs de Aplicación (DTOs internos: Inputs/Outputs de Casos de Uso)

`<source-root>` es `src` si el proyecto usa `src/`; si no, es la raíz del repositorio.

### Cuándo se usa
- Como "comando" o "consulta" de un caso de uso: parámetros de entrada y, opcionalmente, su resultado.
- En el límite **Presentación ↔ Aplicación** (no en el límite con el mundo externo).

### Dónde se define
- `<source-root>/modules/<feature>/application/commands` — **inputs** para casos de uso de escritura (CreateUserInput, UpdateUserInput, DeleteUserInput)
- `<source-root>/modules/<feature>/application/queries` — **inputs** para casos de uso de lectura (GetUserByIdQuery, ListUsersQuery)
- `<source-root>/modules/<feature>/application/results` — **outputs** opcionales cuando la respuesta del caso de uso necesita un formato específico para la UI (UserDetailResult, UserListResult) (**Importante:** Estos NO son DTOs de infraestructura — son contratos internos de aplicación que expresan las necesidades de lectura de la UI sin acoplarse a formatos externos.)

> **Nota sobre combinación:** Un caso de uso puede usar **command/query como input** y **result como output** simultáneamente. Por ejemplo: `createUser(input: CreateUserInput): Promise<UserCreatedResult>` o `listUsers(query: ListUsersQuery): Promise<UserListResult[]>`. No es obligatorio tener las tres carpetas; usá solo lo que tu caso de uso requiera.

### Desde dónde se importa ✅
- **Presentación** (Views/Pages/UI) para invocar casos de uso
- **Tests de aplicación**
- **Adaptadores entrantes** de infraestructura (router, efectos) — permitido porque `infrastructure → application` está permitido

### Desde dónde NO se importa ❌
- **Nunca desde dominio** (el dominio no conoce la aplicación)
- **Nunca desde adaptadores salientes** (repos) para no acoplar el puerto a formatos externos

### Para qué se usa
- Expresar el contrato interno del **caso de uso** de forma estable y orientada al negocio/UX.
- Servir como **DTO interno estable**: solo cambia cuando cambia la regla de negocio, no cuando cambian APIs externas, DB o formatos de transporte.
- Validar y modelar requisitos de negocio al **nivel de aplicación** (campos obligatorios, consistencia básica antes de llegar a dominio).

### Por qué se usa
- Evita que los formatos externos contaminen la API del **caso de uso**.
- Facilita testabilidad y evolución de la **UI** sin romper casos de uso.
- Proporciona un punto de validación independiente del **dominio**.

**Ejemplo mínimo:**

```ts
// <source-root>/modules/users/application/commands/CreateUserInput.ts
export interface CreateUserInput {
  name: string;
  email: string;
}
```

---

## DTOs de Infraestructura (DTOs externos: Contratos Externos)

### Cuándo se usa
- En el límite **Infraestructura ↔ Mundo externo**: `HTTP`/`GraphQL`/`SDK`/`Storage`/`IndexedDB`
- Para requests y responses "crudos" que requieren mapeo hacia/desde el dominio

### Dónde se define
- `<source-root>/modules/<feature>/infrastructure/api/dto`
- `<source-root>/modules/<feature>/infrastructure/api/dto/mapper.ts` (lógica de conversión)

### Desde dónde se importa ✅
- **Solo desde infraestructura**: api/gateways/adapters/repositorios

### Desde dónde NO se importa ❌
- **Nunca desde aplicación**
- **Nunca desde dominio**
- **Nunca desde UI/Presentación**

### Para qué se usa
- Tipar el contrato de red/SDK y centralizar parseo/mapeo.
- Servir como **DTO externo variable**: cambia cuando cambia la API externa, el esquema de la base de datos o el contrato HTTP con clientes.
- Saneamiento y validación de datos externos antes de cruzar hacia el núcleo.

### Por qué se usa
- Aísla formatos externos cambiantes de tu modelo interno (**dominio**/**aplicación**).
- Permite mapear inconsistencias (snake_case, opcionales, nullables) a entidades/ValueObjects estables.
- Protege el núcleo de cambios en APIs externas.

**Ejemplo mínimo:**

```ts
// <source-root>/modules/users/infrastructure/api/dto/UserDto.ts
export interface UserDto {
  id: string;
  name: string;
  email: string;
}

// <source-root>/modules/users/infrastructure/api/dto/mapper.ts
export const toDomain = (dto: UserDto): User => {
  return new User(dto.id, dto.name, dto.email);
};
```

---

## Reglas Prácticas (Cheat Sheet)

| Capa                                 | Puede Importar                   | No Puede Importar                |
| ------------------------------------ | -------------------------------- | -------------------------------- |
| **Presentation**                     | Casos de uso, DTOs de aplicación | DTOs de infraestructura          |
| **Application (Use Cases)**          | Puertos de `domain` (interfaces), Entidades/VO de `domain` | DTOs de infraestructura          |
| **Infrastructure (Adapters/Repos)**  | Puertos, `domain`, DTOs de `infrastructure` | —                                |
| **Domain**                           | Solo otros módulos de `domain`   | Cualquier cosa fuera de `domain` |

### Flujo de conversión

```
Mundo Externo → DTO externo (`infrastructure`) → Mapper → Entidad/ValueObject de `domain` → Caso de Uso → DTO interno (`application`) → UI
```

### Pregunta de decisión rápida

- **"¿Esto modela el contrato con la UI/caso de uso, y quiero que sea estable frente a cambios de infraestructura?"**  
  → Aplicación (`commands`/`queries`/`results`) → DTO interno.

- **"¿Esto modela el contrato con API/SDK/Storage y puede cambiar si cambia el mundo exterior?"**  
  → Infraestructura (`api/dto` + `mapper`) → DTO externo.
