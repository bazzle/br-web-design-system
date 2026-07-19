# Design System

A small design system consumed by a personal portfolio and a couple of other Next.js sites. Currently being rebuilt on proper foundations.

## Core idea

`tokens/` JSON is the **single source of truth**. Everything else — CSS, Figma variables, docs — is generated from it.

```
tokens.json  →  Style Dictionary  →  ├─ CSS custom properties
                                     ├─ Figma variables payload
                                     └─ Storybook token docs
```

Tokens are tiered: **primitive** (raw values) → **semantic** (meaning) → **component** (specific bindings, used sparingly). Components reference the semantic tier, never primitives directly. Format is [DTCG](https://www.designtokens.org/) (`$value`, `$type`, `$description`).

## Structure

```
design-system/
├── tokens/       # JSON source of truth (DTCG)
├── build/        # Style Dictionary config + custom transforms
├── dist/
│   ├── css/      # generated, gitignored
│   └── figma/    # generated, committed (diffs reviewable in PRs)
├── components/   # React components, shipped as untranspiled source
├── .storybook/
└── package.json
```

```bash
# npm install
# npm run build     # tokens → dist
# npm run storybook
```

## Consuming

Published to npm as a scoped package and installed as a versioned dependency. Because `components/` ships untranspiled, consuming Next.js sites need:

```js
// next.config.js
module.exports = { transpilePackages: ['@scope/design-system'] }
```
