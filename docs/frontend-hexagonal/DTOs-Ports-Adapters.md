# DTOs, Puertos y Adaptadores

Guía práctica para ubicar DTOs, definir puertos y escribir adaptadores en un frontend con arquitectura hexagonal y Screaming Architecture.

## Regla de oro

- Los DTOs **externos** representan contratos con el mundo exterior (HTTP/SDK/storage, filas de DB, etc.) y viven en la capa `infrastructure`.
- La aplicación define sus propios contratos **internos** (inputs/comandos/resultados de casos de uso) y NO depende de `infrastructure`.
- Los puertos (interfaces) viven en `domain`; las implementaciones concretas (adaptadores) viven en `infrastructure`.
- La capa `domain` es pura (entities/VO/rules) y no conoce ni `application` ni `infrastructure`.

La clave no es tanto **DTO de entrada vs DTO de salida**, sino **DTO interno vs DTO externo**:

- DTO interno (aplicación/dominio) → contrato estable, definido por tu negocio/casos de uso.
- DTO externo (infraestructura) → contrato variable, definido por APIs/DB/clientes.

Dirección de dependencias permitida:

```
infrastructure  →  application  →  domain
```

Nunca: `application → infrastructure` ni `domain → (application|infrastructure)`.

Más detalle en: `docs/Reglas-de-Dependencias.md`.

---

## Ubicación por capas (sugerida)

```
src/modules/<feature>/
├── domain/
│   ├── entities/
│   ├── value-objects/
│   └── repositories/          # interfaces (Repository, Services)
├── application/
│   ├── use-cases/
│   ├── commands/              # inputs internos de casos de uso
└── infrastructure/
    ├── api/                   # funciones HTTP/SDK
    │   ├── dto/               # DTOs externos (contratos con el mundo de afuera)
    │   └── ...
    ├── adapters/ o repositories/
    │   └── ...                # implementaciones de puertos
    └── gateways/ (opcional)   # cliente de red/SDK, bajo nivel
```

### Responsabilidades

- `domain/repositories`: interfaces que expresan lo que la app necesita. Ej.: `UserRepository`.
- `infrastructure/repositories`: adaptadores que implementan puertos. Ej.: `UserRepositoryFetch`.
- `infrastructure/api`: funciones que hablan con el exterior (HTTP/SDK). Ej.: `createUser()`.
- `infrastructure/api/dto`: formas de datos **externas**. Ej.: `UserDto`, `CreateUserDto`.
- `infrastructure/api/dto/mapper.ts`: transforma DTO ↔ entidades del `domain`.
- `application/commands`: inputs internos para casos de uso (DTOs internos). Ej.: `CreateUserInput`.
- `domain`: entidades/VO/reglas puras. Ej.: `User`, `Email`.

---

## ¿Por qué los DTOs van en `infrastructure`?

Porque describen contratos que no controlás (API, SDK, storage). Su forma puede cambiar y no debe “contaminar” tu lenguaje de `domain` ni la API interna de tus casos de uso. La capa `infrastructure` los convierte a modelos del `domain` (o a inputs internos) mediante mappers.

Ejemplo: backend devuelve snake_case, campos extra u opcionales. El adaptador mapea eso al modelo estable del `domain`.

---

## ¿Puertos en `domain` o `application`?

En `domain`. Los puertos (interfaces) son parte del lenguaje del `domain` y expresan qué necesita la lógica de negocio para cumplir su trabajo. Para puertos orientados al negocio usá `domain/repositories`; para capacidades externas genéricas (HTTP client, DB client, message broker) usá `domain/services`. La implementación concreta se resuelve fuera (`infrastructure`/ composición).

```ts
// domain/repositories/UserRepository.ts
export interface UserRepository {
  create(name: string, email: string): Promise<User>;
}
```

---

## ¿El adaptador puede importar `domain`?

Sí. Las capas externas pueden depender de las internas. Es correcto que un repositorio en `infrastructure` construya `User` a partir de un `UserDto`. Lo prohibido es lo inverso: `domain` y `application` no pueden importar `infrastructure`.

---

## Separar `api/createUser.ts` del repository: ¿cuándo?

- Separar (recomendado) cuando querés:
  - Testear la llamada HTTP de forma aislada.
  - Reutilizar la función desde otros adaptadores (SSR, jobs, etc.).
  - Cambiar cliente (fetch/axios/SDK) sin tocar el repository.
- Integrar en el repo (válido) si buscás máxima simplicidad y no hay reutilización.

Patrón recomendado (separado):

```
infrastructure/api/createUser.ts   # hace la request
infrastructure/api/dto/mapper.ts   # mapea DTO → domain
infrastructure/repositories/...    # implementa el puerto usando lo anterior
```

---

## Convenciones de nombres

- Puerto: `UserRepository` (`domain/repositories`)
- Adaptador: `UserRepositoryFetch` (`infrastructure/repositories`)
- DTOs: `UserDto`, `CreateUserDto` (`infrastructure/api/dto`)
- Mapper: `dtoToUser`, `userToDto` (`infrastructure/api/dto/mapper.ts`)
- Comando de caso de uso: `CreateUserInput` (`application/commands`)

---

## Testing

- Domain: unit tests puros de entidades/VO.
- Application: unit tests de casos de uso con dobles de puertos (mocks/stubs).
- Infrastructure: tests de integración de adaptadores (mapeos y llamadas reales o simuladas a `fetch`).

---

## Antipatrones a evitar

- Importar DTOs de `infrastructure` desde `application` o `domain`.
- Reutilizar un `User` genérico en `shared` para todos los contextos (termina con muchos opcionales y invariantes débiles).
- Hacer que `domain` conozca `fetch`, `axios`, `localStorage` o el formato JSON externo.

---

## Recursos relacionados

- Reglas de importación y ejemplo ESLint: `docs/Reglas-de-Dependencias.md`
- Ejemplo completo CreateUser: `docs/frontend-hexagonal/examples/CreateUser.md`
- DTOs de `application` vs `infrastructure` (cuándo/desde dónde/por qué): `docs/frontend-hexagonal/DTOs-Aplicacion-vs-Infraestructura.md`
- Ejemplo de lectura (GetUsers): `docs/frontend-hexagonal/examples/GetUsers.md`
- Repositorios: contratos de retorno: `docs/Repositorios-Contratos-y-Retornos.md`
