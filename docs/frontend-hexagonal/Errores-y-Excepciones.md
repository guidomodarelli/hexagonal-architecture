# Manejo de Errores

Regla general: los errores se originan en la capa donde se detecta la condición anómala, pero se expresan con el vocabulario adecuado de cada capa.

`<source-root>` es `src` si el proyecto usa `src/`; si no, es la raíz del repositorio.

- Dominio: errores de negocio (invariantes, autorizaciones por reglas de negocio, validaciones semánticas).
- Application / Casos de uso: coordinan colaboradores, lanzan o propagan errores del dominio y dejan la traducción HTTP/UI para el borde.
- Infraestructura / Adaptadores: errores técnicos (HTTP, red, timeouts, DB, serialización) envueltos en errores del puerto.
- Entrypoints (API, CLI, UI): traducen errores internos al contrato externo (HTTP status, mensajes de UI, etc.).

## Decisiones por capa

### Dominio y Casos de Uso
- Definen y lanzan errores de negocio (p. ej. `UserNotAuthorizedToSearchError`).
- No conocen detalles de transporte (HTTP) ni de proveedores (OpenSearch, SQL, etc.).
- Propagan errores técnicos tal como llegan desde el puerto (sin convertirlos en negocio).
- Solo transforman errores si el contrato **interno** del caso de uso realmente necesita otro resultado de aplicación; no traducen a HTTP ni a mensajes de UI.

### Infraestructura / Adaptadores
- Capturan errores de proveedores (HTTP 403/500, SDKs, drivers) y los **traducen** a errores del puerto (p. ej. `SearchRepositoryPermissionError`, `SearchRepositoryUnavailableError`).
- No inventan errores de negocio.
- Evitan filtrar detalles de librerías externas (nombres de clases, códigos específicos, etc.).

```typescript
// Controlador - ❌ MAL EJEMPLO
export const searchController = async (req, res) => {
  try {
    const result = await searchUseCase.execute(req.body);
    res.json(result);
  } catch (error) {
    // ❌ Filtra TODO al usuario - PELIGROSO
    res.status(500).json({
      message: error.message, // Puede exponer: "Database connection failed on server db-prod-01"
      stack: error.stack,     // Expone rutas del servidor, código interno
      details: error.details  // Puede exponer estructura interna
    });
  }
};

// Controlador - ✅ BUEN EJEMPLO
export const searchController = async (req, res) => {
  try {
    const result = await searchUseCase.execute(req.body);
    res.json(result);
  } catch (error) {
    // ✅ Traduce sin exponer detalles internos
    const response = translateErrorToHttp(error);
    res.status(response.status).json({
      message: response.message,
      code: response.code
    });
  }
};

function translateErrorToHttp(error) {
  // Solo expone información segura y útil para el cliente
  switch (error.constructor.name) {
    case 'UserNotFoundError':
      return {
        status: 404,
        message: 'Usuario no encontrado',
        code: 'USER_NOT_FOUND'
      };

    case 'InvalidPermissionsError':
      return {
        status: 403,
        message: 'No tienes permisos para esta acción',
        code: 'FORBIDDEN'
      };

    case 'DatabaseConnectionError':
      // ❌ NO exponer: "Failed to connect to MongoDB at 192.168.1.100:27017"
      // ✅ SÍ exponer: mensaje genérico
      return {
        status: 500,
        message: 'Error interno del servidor',
        code: 'INTERNAL_ERROR'
      };

    default:
      return {
        status: 500,
        message: 'Error interno del servidor',
        code: 'INTERNAL_ERROR'
      };
  }
}
```

### Entrypoints (API/Controladores o UI)
- Traducen errores a un contrato externo:
  - API REST: `UserNotAuthorizedToSearchError` → 403; indisponibilidad técnica → 503.
  - UI (React/Vue): muestran mensajes adecuados ("No tenés permisos" vs "Servicio no disponible").

## Caso práctico: permisos sobre un índice (OpenSearch)

Contexto: consultamos un índice en OpenSearch vía HTTP. A veces el backend responde `403 Forbidden`.

- Si el 403 refleja una **regla de negocio** ("este usuario no puede buscar en ese índice"), el **caso de uso** debe lanzar un error de dominio.
- Si el 403 refleja un **problema técnico** (roles/credenciales del sistema mal configurados), es un **error del adaptador** (infraestructura) que el puerto expone como `SearchRepositoryPermissionError`.

**¿Cómo te das cuenta de la diferencia?**

1. **Timing**: ¿Cuándo sabés que el usuario no tiene permiso?
  - **Antes de llamar al backend** → Es regla de negocio. Chequeás roles/permisos del usuario en el `AuthorizationPolicy` y lanzás `UserNotAuthorizedToSearchError`.
  - **Después de que el backend responde 403** → Es error técnico. Las credenciales del sistema están mal o el backend tiene una configuración incorrecta.

2. **Responsabilidad**: ¿Quién decide el permiso?
  - **Tu aplicación** (caso de uso + policy) → Dominio/negocio.
  - **El proveedor externo** (OpenSearch, API de terceros) → Infraestructura/técnico.

3. **Solución**: ¿Cómo se arregla?
  - **Negocio**: El usuario necesita más permisos (cambio de rol, autorización).
  - **Técnico**: DevOps/Admin debe reconfigurar credenciales del sistema, roles en OpenSearch, tokens, etc.

4. **Código**: ¿Dónde ocurre?
  - **Negocio**: Validación en el `UseCase` antes de `repo.search()`.
  - **Técnico**: Captura del status HTTP 403 dentro del adaptador (`OpenSearchSearchRepository`).

**Ejemplo práctico:**
- Usuario común intenta buscar en índice "confidencial" → `AuthorizationPolicy.canSearchIndex()` retorna `false` → Caso de uso lanza `UserNotAuthorizedToSearchError` (403 en API).
- Sistema intenta buscar con token válido pero OpenSearch responde 403 porque el rol del service account no tiene acceso → Adaptador captura y lanza `SearchRepositoryPermissionError` (503 en API para indicar problema del sistema).

### Tipos y puertos

```ts
// <source-root>/modules/search/domain/repositories/SearchRepository.ts
export type SearchQuery = unknown;
export type SearchResult = { id: string; source: unknown };

export interface SearchRepository {
  search(params: { index: string; query: SearchQuery }): Promise<SearchResult[]>;
}

// Errores técnicos del puerto
export class SearchRepositoryError extends Error { readonly scope = 'infrastructure'; }
export class SearchRepositoryPermissionError extends SearchRepositoryError {
  constructor(public readonly index: string) {
    super(`Forbidden on index ${index}`);
  }
}
export class SearchRepositoryUnavailableError extends SearchRepositoryError {}
```

### Errores de dominio

```ts
// domain/errors.ts
export class DomainError extends Error { readonly scope = 'domain'; }

export class UserNotAuthorizedToSearchError extends DomainError {
  constructor(public readonly userId: string, public readonly index: string) {
    super(`User ${userId} is not allowed to search index ${index}`);
  }
}
```

### Caso de uso

```ts
// application/policies/authorization-policy.ts
export type User = { id: string; roles: string[] };
export interface AuthorizationPolicy {
  canSearchIndex(user: User, index: string): Promise<boolean> | boolean;
}
```

Para mantener el foco en el manejo de errores, el ejemplo de abajo usa una clase. La convención preferida de este repo para frontend sigue siendo `searchInIndex(deps)(params)`, pero la frontera entre error de dominio y error técnico no cambia.

```ts
// application/use-cases/search-in-index.ts
import {
  SearchRepository,
  SearchRepositoryError,
  SearchRepositoryPermissionError,
  SearchResult,
} from '../../domain/repositories/SearchRepository';
import { AuthorizationPolicy, User } from '../policies/authorization-policy';
import { UserNotAuthorizedToSearchError } from '../../domain/errors';

export class SearchInIndexUseCase {
  constructor(
    private readonly repo: SearchRepository,
    private readonly policy: AuthorizationPolicy
  ) {}

  async execute(params: { user: User; index: string; query: unknown }): Promise<SearchResult[]> {
    const { user, index, query } = params;

    // 1) Regla de negocio antes de tocar infraestructura
    const allowed = await this.policy.canSearchIndex(user, index);
    if (!allowed) throw new UserNotAuthorizedToSearchError(user.id, index);

    // 2) Llamada al puerto; propagamos errores técnicos sin convertir a negocio
    try {
      return await this.repo.search({ index, query });
    } catch (err) {
      if (err instanceof SearchRepositoryError) throw err; // técnico
      throw err; // inesperado
    }
  }
}
```

### Adaptador de OpenSearch (`infrastructure`)

```ts
// <source-root>/modules/search/infrastructure/opensearch/OpenSearchSearchRepository.ts
import {
  SearchRepository,
  SearchRepositoryPermissionError,
  SearchRepositoryUnavailableError,
  SearchRepositoryError,
  SearchResult,
} from '../../domain/repositories/SearchRepository';

type Http = (url: string, init: RequestInit) => Promise<Response>;

export class OpenSearchSearchRepository implements SearchRepository {
  constructor(
    private readonly http: Http,
    private readonly baseUrl: string,
    private readonly authHeader?: string
  ) {}

  async search(params: { index: string; query: unknown }): Promise<SearchResult[]> {
    const { index, query } = params;

    const res = await this.http(
      `${this.baseUrl}/${encodeURIComponent(index)}/_search`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          ...(this.authHeader ? { Authorization: this.authHeader } : {}),
        },
        body: JSON.stringify(query),
      }
    ).catch((e) => {
      throw new SearchRepositoryUnavailableError(String(e?.message ?? e));
    });

    if (res.status === 403) throw new SearchRepositoryPermissionError(index);
    if (res.status >= 500) throw new SearchRepositoryUnavailableError(`OpenSearch ${res.status}`);
    if (!res.ok) throw new SearchRepositoryError(`OpenSearch ${res.status}`);

    const data = await res.json();
    const hits = data?.hits?.hits ?? [];
    return hits.map((h: any) => ({ id: h._id, source: h._source })) as SearchResult[];
  }
}
```

### Traducción en el borde

Podés tener dos tipos de entrypoints. Un ejemplo para API (BFF/Node) y otro para UI.

API/Controlador (Node/Express)
```ts
// interface/http/search-controller.ts
import { Request, Response } from 'express';
import { SearchInIndexUseCase } from '../../application/use-cases/search-in-index';
import {
  SearchRepositoryPermissionError,
  SearchRepositoryUnavailableError,
} from '../../domain/repositories/SearchRepository';
import { UserNotAuthorizedToSearchError } from '../../domain/errors';

export class SearchController {
  constructor(private readonly useCase: SearchInIndexUseCase) {}

  handle = async (req: Request, res: Response) => {
    try {
      const results = await this.useCase.execute({
        user: req.user as any,
        index: req.params.index,
        query: req.body,
      });
      return res.status(200).json(results);
    } catch (err) {
      if (err instanceof UserNotAuthorizedToSearchError) {
        return res.status(403).json({ code: 'USER_NOT_AUTHORIZED', message: err.message });
      }
      if (err instanceof SearchRepositoryPermissionError) {
        return res.status(503).json({
          code: 'SEARCH_BACKEND_FORBIDDEN',
          message: 'Search backend not authorized. Check system credentials/roles.',
        });
      }
      if (err instanceof SearchRepositoryUnavailableError) {
        return res.status(503).json({ code: 'SEARCH_BACKEND_UNAVAILABLE', message: 'Search backend unavailable.' });
      }
      return res.status(500).json({ code: 'INTERNAL_ERROR' });
    }
  };
}
```

UI (React)
```tsx
// presentation/pages/SearchPage.tsx
import { useState } from 'react';
import { SearchInIndexUseCase } from '../../application/use-cases/search-in-index';
import { UserNotAuthorizedToSearchError } from '../../domain/errors';
import { SearchRepositoryError, SearchRepositoryPermissionError } from '../../domain/repositories/SearchRepository';

export function SearchPage({ useCase }: { useCase: SearchInIndexUseCase }) {
  const [message, setMessage] = useState<string | null>(null);

  const onSearch = async () => {
    setMessage(null);
    try {
      const results = await useCase.execute({ user: { id: 'u1', roles: [] }, index: 'idx', query: {} });
      console.log(results);
    } catch (err) {
      if (err instanceof UserNotAuthorizedToSearchError) {
        setMessage('No tenés permisos para buscar en este índice.');
        return;
      }
      if (err instanceof SearchRepositoryPermissionError) {
        setMessage('El buscador no está autorizado. Contactá a soporte.');
        return;
      }
      if (err instanceof SearchRepositoryError) {
        setMessage('Servicio de búsqueda no disponible.');
        return;
      }
      setMessage('Ocurrió un error inesperado.');
    }
  };

  return (
    <div>
      <button onClick={onSearch}>Buscar</button>
      {message && <p role="alert">{message}</p>}
    </div>
  );
}
```

## Regla práctica (resumen)
- Negocio/regla de permisos de usuarios → error de dominio en el caso de uso.
- Problema técnico del backend (roles/credenciales del sistema) → error de infraestructura del puerto.
- Entrypoints traducen a HTTP/UI; no mezclar semánticas.

## Checklist rápido
- ✅ El dominio lanza errores semánticos (no HTTP)
- ✅ Los casos de uso coordinan y propagan; no traducen a HTTP/UI
- ✅ Los adaptadores traducen errores externos a errores del puerto
