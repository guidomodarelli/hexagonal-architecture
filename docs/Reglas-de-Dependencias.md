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
  - `modules/setup.[tj]s` → `modules/<feature>/setup.[tj]s`
  - shell externo / tests de integración → `modules/<feature>/setup.[tj]s`
  - shell externo / tests de integración → `modules/setup.[tj]s`
- Prohibido:
  - `domain/*` → `application/*`, `presentation/*` o `infrastructure/*`
  - `application/*` → `presentation/*` o `infrastructure/*`
  - `presentation/*` → `infrastructure/*`
  - `infrastructure/*` → `presentation/*`
  - `presentation/*`, `application/*`, `domain/*` o `infrastructure/*` → `setup.[tj]s`

## Estructura sugerida por módulo

```
<source-root>/modules/
├── setup.ts         # agregador o factory opcional
└── <feature>/
    ├── setup.ts     # builder del feature
    ├── presentation/   # opcional en frontend o adapters entrantes
    ├── domain/
    ├── application/
    └── infrastructure/
```

`<source-root>` es `src` si el proyecto usa `src/`; si no, es la raíz del repositorio.

## `setup.[tj]s` como frontera de composición

`modules/setup.[tj]s` es opcional. Cuando existe, puede centralizar dependencias compartidas una sola vez por boundary externo, importar los builders `build<Feature>Module(...)` desde `modules/<feature>/setup.[tj]s` y devolver un objeto `modules.<feature>.useCases.*` o factories lazy por feature.

`modules/<feature>/setup.[tj]s` vive cerca del feature, pero no forma parte de `presentation`, `application`, `domain` ni `infrastructure`. Su responsabilidad es declarar `build<Feature>Module(deps)` con dependencias explícitas; las carpetas internas no deberían importarlo a él.

- El shell externo del framework, tests de integración u otros entrypoints pueden consumir `modules/setup.[tj]s` o importar solo `modules/<feature>/setup.[tj]s` para componer un feature puntual.
- Evitá usar `modules/setup.[tj]s` o `modules/<feature>/setup.[tj]s` como service locator compartido desde `presentation`, `application`, `domain` o `infrastructure`.

## Archivos externos al módulo

Archivos de routing o bootstrap del framework como `app/routes/...`, `src/pages/...` o `main.tsx` viven fuera de `modules/*`. No pertenecen a `presentation`, `application`, `domain` ni `infrastructure`: forman el shell externo de la app.

- Pueden importar `modules/setup.[tj]s` o `modules/<feature>/setup.[tj]s`, junto con `modules/<feature>/presentation/...`, para conectar una ruta con su View.
- No deberían contener lógica de negocio ni implementaciones concretas de adaptadores; solo wiring.
- Si usan `modules/setup.[tj]s`, conviene que sea un agregador liviano o factories lazy por feature para no cargar adaptadores de módulos no usados en apps con code splitting por ruta.

## Enforzar con ESLint (opcional)

Ejemplo con `eslint-plugin-import` (`import/no-restricted-paths`) usando paths dentro de `src/modules/`.
Si tu proyecto no usa `src/`, reemplazá `./src/modules` por `./modules`:

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

          // inner folders cannot import any composition root
          { target: './src/modules/**/domain', from: './src/modules/setup.*' },
          { target: './src/modules/**/application', from: './src/modules/setup.*' },
          { target: './src/modules/**/presentation', from: './src/modules/setup.*' },
          { target: './src/modules/**/infrastructure', from: './src/modules/setup.*' },
          { target: './src/modules/**/domain', from: './src/modules/**/setup.*' },
          { target: './src/modules/**/application', from: './src/modules/**/setup.*' },
          { target: './src/modules/**/presentation', from: './src/modules/**/setup.*' },
          { target: './src/modules/**/infrastructure', from: './src/modules/**/setup.*' },

          // feature setup cannot import the root setup
          { target: './src/modules/*/setup.*', from: './src/modules/setup.*' },
        ],
      },
    ],
  },
};
```

Alternativa con `eslint-plugin-boundaries` (zonas por carpeta):
Si tu proyecto no usa `src/`, reemplazá `src/modules` por `modules` en los `pattern`:

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
      { type: 'setup-root', pattern: 'src/modules/setup.*' },
      { type: 'setup-feature', pattern: 'src/modules/*/setup.*' },
    ],
  },
  rules: {
    'boundaries/element-types': [
      'error',
      {
        default: 'allow',
        rules: [
          { from: 'domain', disallow: ['presentation', 'application', 'infrastructure', 'setup-root', 'setup-feature'] },
          { from: 'application', disallow: ['presentation', 'infrastructure', 'setup-root', 'setup-feature'] },
          { from: 'presentation', disallow: ['infrastructure', 'setup-root', 'setup-feature'] },
          { from: 'infrastructure', disallow: ['presentation', 'setup-root', 'setup-feature'] },
          { from: 'setup-feature', disallow: ['setup-root'] },
        ],
      },
    ],
  },
};
```

Nota: Los fragmentos son ejemplos; ajustá paths y alias según tu proyecto.
