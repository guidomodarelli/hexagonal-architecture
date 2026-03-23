# Reglas de Dependencias

Estas reglas concretan la dirección de dependencias y qué importaciones están permitidas entre capas.

## Dirección de dependencias

```
presentation    →  application
presentation    →  domain        (opcional, solo para tipos/validaciones puras)
infrastructure  →  application
infrastructure  →  domain
application     →  domain
```

- La regla correcta no es "cada capa solo conoce a la inmediatamente inferior".
- La regla correcta es: **toda dependencia apunta hacia adentro, nunca hacia afuera**.
- `Domain` no importa nada de `application`, `presentation` ni `infrastructure`.
- `Application` importa `domain`, pero no importa `presentation` ni `infrastructure`.
- `Presentation` puede importar `application` para ejecutar casos de uso y, si aporta claridad, piezas puras de `domain` como tipos o validaciones.
- `Infrastructure` puede importar `application` (para conectar adaptadores entrantes con casos de uso) y `domain` (para construir entidades/Value Objects e implementar puertos).

## Matriz de importaciones

- Permitido:
  - `presentation/*` → `application/*`
  - `presentation/*` → `domain/*` (solo tipos, Value Objects o validaciones puras)
  - `application/*` → `domain/*`
  - `infrastructure/*` → `application/*`
  - `infrastructure/*` → `domain/*`
- Prohibido:
  - `domain/*` → `application/*`, `presentation/*` o `infrastructure/*`
  - `application/*` → `presentation/*` o `infrastructure/*`
  - `presentation/*` → `infrastructure/*`
  - `infrastructure/*` → `presentation/*`

## Estructura sugerida por módulo

```
src/modules/<feature>/
├── presentation/   # opcional en frontend o adapters entrantes
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
          // domain cannot import presentation/application/infrastructure
          { target: './src/modules/**/domain', from: './src/modules/**/presentation' },
          { target: './src/modules/**/domain', from: './src/modules/**/application' },
          { target: './src/modules/**/domain', from: './src/modules/**/infrastructure' },

          // application cannot import presentation/infrastructure
          { target: './src/modules/**/application', from: './src/modules/**/presentation' },
          { target: './src/modules/**/application', from: './src/modules/**/infrastructure' },

          // presentation cannot import infrastructure
          { target: './src/modules/**/presentation', from: './src/modules/**/infrastructure' },

          // infrastructure cannot import presentation
          { target: './src/modules/**/infrastructure', from: './src/modules/**/presentation' },
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
      { type: 'presentation', pattern: 'src/modules/*/presentation/**' },
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
          { from: 'domain', disallow: ['presentation', 'application', 'infrastructure'] },
          { from: 'application', disallow: ['presentation', 'infrastructure'] },
          { from: 'presentation', disallow: ['infrastructure'] },
          { from: 'infrastructure', disallow: ['presentation'] },
        ],
      },
    ],
  },
};
```

Nota: Los fragmentos son ejemplos; ajustá paths y alias según tu proyecto.
