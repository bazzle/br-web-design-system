# CLAUDE.md

Context for Claude Code working in this design system repo.

## What this is

A design system consuming by a personal portfolio site and a couple of other
sites, all Next.js. Currently sparse — a few global CSS files defining colours
and font sizes, pulled into consuming sites as git submodules. We are rebuilding
it on proper foundations.

This is a learning project as well as a working system. Prefer conventional,
well-documented approaches over clever ones, and explain the reasoning behind
structural choices rather than just applying them. Where a convention has a name
(token tiers, semver, DTCG), use the name.

## Core architecture decision

**`tokens/` JSON is the single source of truth. Everything else is generated
from it. The flow is one-directional.**

```
tokens.json  →  Style Dictionary  →  ├─ CSS custom properties (sites, components)
                                     ├─ Figma variables payload (design tool)
                                     └─ Storybook token docs
```

Bidirectional sync between Figma and code was explicitly rejected. Two editable
origins means merge conflicts in a system with no merge tooling. Figma is a
**downstream consumer**, never authoritative.

Code wins as origin because it is versioned, diffable, and reviewable in PRs.

## Repo structure

```
design-system/
├── tokens/           # JSON source of truth (DTCG format)
├── build/            # Style Dictionary config + custom transforms
├── dist/
│   ├── css/          # generated — gitignored, rebuilt on publish
│   └── figma/        # generated — COMMITTED, so token diffs are reviewable
├── components/       # React components, shipped as untranspiled source
├── .storybook/
└── package.json
```

`dist/css/` is gitignored deliberately. If generated CSS sits in the repo,
someone eventually edits it directly and the source of truth silently diverges.

`dist/figma/` is the exception — it is not consumed by npm at all, only by the
Figma sync step. Committing it means a PR shows "this changes 4 Figma variables",
which is useful in review.

## Token tiers

Tokens are tiered. This is the central concept of the system, not a detail.

- **Primitive** — raw values. `color.blue.500`. Says nothing about usage.
- **Semantic** — meaning. `semantic.action.primary` → `{color.blue.500}`.
- **Component** — specific bindings. Often unnecessary at this scale; add only
  when a component genuinely needs to diverge.

**Components must reference the semantic tier, never primitives directly.** That
is what makes theming and rebranding a repoint of one layer rather than a
find-and-replace across the codebase.

Aliasing uses DTCG curly-brace syntax: `"$value": "{color.blue.500}"`.

Naming the semantic tier is the hard, judgement-heavy part. Flag naming choices
for discussion rather than deciding unilaterally.

## Token format

DTCG (Design Tokens Community Group) format — `$value`, `$type`, `$description`.
The spec reached its first stable version (2025.10) in October 2025.

Note: Style Dictionary has had first-class DTCG support since v4, but full
2025.10 support is still in progress in v5. Check current support before relying
on newer spec features.

## Fluid typography

Fluid type is stored as its **parts**, not as a pre-baked `clamp()` string:

```json
"heading-small": {
  "$type": "fluidSize",
  "$value": { "min": "2.09rem", "fluid": "2.2vw", "max": "2.2rem" }
}
```

- The CSS transform assembles these into `clamp(min, fluid, max)`.
- The Figma transform emits only `min` and `max` as two variable modes, because
  Figma cannot represent continuous viewport-based scaling at all.

This is the clearest demonstration of why the token layer exists: one token, two
platform-appropriate outputs. Storybook is where fluid behaviour can actually be
observed, since it renders real CSS in a resizable frame.

## Packaging

Published to npm as a scoped package, consumed by sites as a versioned
dependency. **Git submodules are being removed** — detached HEADs, no meaningful
version pinning, teammates forgetting `--recursive`.

```json
{
  "files": ["dist", "components"],
  "publishConfig": { "access": "public" },
  "scripts": { "prepublishOnly": "npm run build" }
}
```

- `files` keeps `tokens/`, `build/`, `.storybook/` and `dist/figma/` out of the
  published tarball. Consumers don't need them.
- Scoped packages default to private visibility, hence `publishConfig`.
- Versions are immutable once published. Treat every publish as permanent.
- Run `npm publish --dry-run` before a first publish to verify what ships.

Semver matters here: a site can sit on `2.1.0` while another stays on `1.x`
through a breaking change.

## Consuming sites

All consumers are Next.js. This is confirmed, not an assumption.

Because `components/` ships as untranspiled source, each consuming site needs:

```js
// next.config.js
module.exports = { transpilePackages: ['@scope/design-system'] }
```

Components using hooks or event handlers need `'use client'` in the package
itself. Purely presentational components stay server components — that is the
preferred default, so only mark the genuinely interactive ones.

## Figma

Downstream consumer, for mockups and ideation only.

Sync happens via **Tokens Studio or a custom Figma plugin** — *not* the Variables
REST API, which requires a Full seat in an Enterprise org and is out of reach
here. The Plugin API has no such restriction.

Figma will never represent fluid typography properly. That is accepted, not a
problem to solve. It gets the min and max endpoints as variable modes.

## Storybook

The living style guide and the primary development surface. Already set up.

Should document both components and tokens — a token reference page
auto-generated from the JSON (using `$description` fields) is a good early win
and one of the concrete arguments for the JSON layer over hand-written CSS.

## Order of work

1. **Extract tokens to JSON, generate the existing CSS from them.** Nothing
   visible changes — that is the success criterion. ← current focus
2. Replace submodules with the published package.
3. Wire Storybook to document tokens and components.
4. Add the Figma sync.

Step 4 is the most interesting and therefore the most tempting to do first. It is
worthless without 1–3, since there would be nothing coherent to push.

## Things not to do

- Do not edit anything in `dist/css/` — it is generated.
- Do not add values directly to CSS files. They go in `tokens/`.
- Do not make Figma authoritative for any value.
- Do not reach for the Figma Variables REST API.
- Do not add a component tier to tokens unless a real need appears.
- Do not restructure into a monorepo yet. It is the natural next step if the
  system grows (separately versioned `@scope/tokens` and `@scope/components`),
  but the complexity is not currently earned.

## Open questions

- Semantic tier naming — unresolved and worth deliberating over.
- Whether the type scale needs three tiers or two. Probably two.
- Whether `heading-small` is already semantic rather than primitive.