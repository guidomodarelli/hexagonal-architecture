# Reglas de Dependencias

Estas reglas concretan la dirección de dependencias y qué importaciones están permitidas entre capas.

## Dirección de dependencias

```
infrastructure  →  application  →  domain
```

- `Domain` no importa nada de `application` ni `infrastructure`.
- `Application` importa `domain`, pero no importa `infrastructure`.
- `Infrastructure` puede importar `application` (para conectar adaptadores entrantes con casos de uso) y `domain` (para construir entidades/Value Objects e implementar puertos).

## Matriz de importaciones

- Permitido:
  - `application/*` → `domain/*`
  - `infrastructure/*` → `application/*`
  - `infrastructure/*` → `domain/*`
- Prohibido:
  - `domain/*` → `application/*` o `infrastructure/*`
  - `application/*` → `infrastructure/*`

## Estructura sugerida por módulo

```
src/modules/<feature>/
├── domain/
├── application/
└── infrastructure/
```

## Enforzar con ESLint (opcional)

Ejemplo con `eslint-plugin-import` (`import/no-restricted-paths`) usando paths dentro de `src/modules/*`:

```js
// .eslintrc.cjs (fragmento ilustrativo)
module.exports = {
  rules: {
    'import/no-restricted-paths': [
      'error',
      {
        zones: [
          // domain cannot import application/infrastructure
          { target: './src/modules/**/domain', from: './src/modules/**/application' },
          { target: './src/modules/**/domain', from: './src/modules/**/infrastructure' },

          // application cannot import infrastructure
          { target: './src/modules/**/application', from: './src/modules/**/infrastructure' },
        ],
      },
    ],
  },
};
```

Alternativa con `eslint-plugin-boundaries` (zonas por carpeta):

```js
// .eslintrc.cjs (fragmento ilustrativo)
module.exports = {
  plugins: ['boundaries'],
  settings: {
    'boundaries/elements': [
      { type: 'domain', pattern: 'src/modules/*/domain/**' },
      { type: 'application', pattern: 'src/modules/*/application/**' },
      { type: 'infrastructure', pattern: 'src/modules/*/infrastructure/**' },
    ],
  },
  rules: {
    'boundaries/element-types': [
      'error',
      {
        default: 'allow',
        rules: [
          { from: 'domain', disallow: ['application', 'infrastructure'] },
          { from: 'application', disallow: ['infrastructure'] },
        ],
      },
    ],
  },
};
```

Nota: Los fragmentos son ejemplos; ajustá paths y alias según tu proyecto.
