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
  - shell externo / tests de integración → `modules/<feature>/setup.[tj]s`
- Prohibido:
  - `domain/*` → `application/*`, `presentation/*` o `infrastructure/*`
  - `application/*` → `presentation/*` o `infrastructure/*`
  - `presentation/*` → `infrastructure/*`
  - `infrastructure/*` → `presentation/*`
  - `presentation/*`, `application/*`, `domain/*` o `infrastructure/*` → `setup.[tj]s`

## Estructura sugerida por módulo

```
<source-root>/modules/<feature>/
├── setup.ts         # composition root del feature
├── presentation/   # opcional en frontend o adapters entrantes
├── domain/
├── application/
└── infrastructure/
```

`<source-root>` es `src` si el proyecto usa `src/`; si no, es la raíz del repositorio.

## `setup.[tj]s` como frontera de composición

`setup.[tj]s` vive dentro de `modules/<feature>` por cercanía, pero no forma parte de `presentation`, `application`, `domain` ni `infrastructure`. Es el composition root del feature: importa esas carpetas para cablear dependencias, y las carpetas internas no deberían importarlo a él.

- Lo consumen el shell externo del framework, tests de integración u otros entrypoints.
- Evitá usarlo como service locator compartido desde `presentation`, `application`, `domain` o `infrastructure`.

## Archivos externos al módulo

Archivos de routing o bootstrap del framework como `app/routes/...`, `src/pages/...` o `main.tsx` viven fuera de `modules/*`. No pertenecen a `presentation`, `application`, `domain` ni `infrastructure`: forman el shell externo de la app.

- Pueden importar `modules/<feature>/setup.[tj]s` y `modules/<feature>/presentation/...` para conectar una ruta con su View.
- No deberían contener lógica de negocio ni implementaciones concretas de adaptadores; solo wiring.
- Si existe un `modules/setup.[tj]s` global, conviene que sea un agregador liviano o factories lazy por feature para no cargar adaptadores de módulos no usados en apps con code splitting por ruta.

## Enforzar con ESLint (opcional)

Ejemplo con `eslint-plugin-import` (`import/no-restricted-paths`) usando paths dentro de `src/modules/*`.
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

          // inner folders cannot import the composition root
          { target: './src/modules/**/domain', from: './src/modules/**/setup.*' },
          { target: './src/modules/**/application', from: './src/modules/**/setup.*' },
          { target: './src/modules/**/presentation', from: './src/modules/**/setup.*' },
          { target: './src/modules/**/infrastructure', from: './src/modules/**/setup.*' },
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
      { type: 'setup', pattern: 'src/modules/*/setup.*' },
    ],
  },
  rules: {
    'boundaries/element-types': [
      'error',
      {
        default: 'allow',
        rules: [
          { from: 'domain', disallow: ['presentation', 'application', 'infrastructure', 'setup'] },
          { from: 'application', disallow: ['presentation', 'infrastructure', 'setup'] },
          { from: 'presentation', disallow: ['infrastructure', 'setup'] },
          { from: 'infrastructure', disallow: ['presentation', 'setup'] },
        ],
      },
    ],
  },
};
```

Nota: Los fragmentos son ejemplos; ajustá paths y alias según tu proyecto.
