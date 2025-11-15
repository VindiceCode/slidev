# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Slidev is a presentation slides tool for developers. It converts Markdown files into interactive, themable presentations powered by Vue 3 and Vite. The codebase is a monorepo using pnpm workspaces.

## Commands

### Development
```bash
# Install dependencies
pnpm install

# Build all packages (parallel)
pnpm build

# Build with watch mode (development)
pnpm dev

# Run demo presentation
pnpm demo:dev

# Run other demo examples
pnpm demo:composable-vue
pnpm demo:vue-runner
```

### Testing
```bash
# Run tests (vitest)
pnpm test

# Type checking
pnpm typecheck
```

### Linting
```bash
# Lint code
pnpm lint

# Lint and fix
pnpm lint:fix
```

### VSCode Extension Development
```bash
pnpm vscode:dev
```

### Documentation
```bash
# Run docs site locally
pnpm docs

# Build docs
pnpm docs:build
```

### Publishing
```bash
# Version bump (interactive)
pnpm release

# Publish to npm (CI only)
pnpm ci:publish
```

## Monorepo Structure

The project uses pnpm workspaces with `shamefullyHoist: true` for flat dependency resolution. All packages are under `/packages`:

### Core Packages

1. **@slidev/cli** (`packages/slidev/`)
   - Main Node.js package and CLI entry point
   - Contains Vite dev server, build system, and export functionality
   - Orchestrates the entire Slidev system
   - Source: `node/` directory
   - Binary: `bin/slidev.mjs`

2. **@slidev/client** (`packages/client/`)
   - Vue 3 SPA frontend (not published separately)
   - Handles rendering, navigation, layouts, and UI
   - Entry: `main.ts`
   - Uses Vue Router for slide navigation

3. **@slidev/parser** (`packages/parser/`)
   - Markdown parser with modular design
   - Exports: `core`, `fs`, `utils` submodules
   - `core.ts`: Pure parsing logic (no I/O)
   - `fs.ts`: File system operations
   - Handles YAML frontmatter, slide detection, feature detection

4. **@slidev/types** (`packages/types/`)
   - Shared TypeScript definitions
   - Used across all packages

5. **create-slidev** (`packages/create-app/`)
   - Scaffolding tool for new projects
   - Invoked via `npm init slidev`

6. **create-slidev-theme** (`packages/create-theme/`)
   - Scaffolding tool for themes
   - Invoked via `npm init slidev-theme`

7. **@slidev/vscode** (`packages/vscode/`)
   - VSCode extension with Language Server Protocol
   - Uses Volar for Vue/Markdown integration

## Architecture

### Data Flow

```
Markdown file (slides.md)
  ↓
@slidev/parser loads & parses
  ↓
SlidevData structure created
  ↓
Vite plugins transform to virtual modules
  ↓
@slidev/client renders in browser
```

### Parser Architecture

The parser is split into modular components:

- **Core parser** (`core.ts`): Pure parsing logic, no file I/O
  - Detects slides (separated by `---`)
  - Parses YAML frontmatter
  - Detects features (KaTeX, Monaco, Mermaid)
  - Supports slide imports with range syntax (`src: pages.md:1-10`)

- **File system layer** (`fs.ts`): Handles file operations
  - Recursive slide imports
  - File watching
  - Caching

- **Preparser hooks**: Extension points for custom transformations before core parsing

### Vite Plugin System

The CLI package contains ~18 specialized Vite plugins that work together:

1. **slidev:loader** (pre-enforce): Transforms markdown to Vue SFC
2. **slidev:layout-wrapper**: Injects layout component wrapping
3. **slidev:context-injection**: Adds `$slidev` global context via Vue provide/inject
4. Plus others for markdown processing, icons, components, config, HMR, etc.

### Virtual Modules

Slidev creates virtual modules dynamically:
- Individual slides: `__slidev_N.md` patterns
- HTTP endpoints for slide data (GET/POST)
- HMR for live reload

### Theme & Addon System

- Resolved at runtime with version validation
- Themes provide layouts, components, and styles
- Addons extend functionality
- Both are npm packages with specific structure
- CLI recursively resolves theme/addon dependencies

### Layout System

Each slide can specify a layout via frontmatter:
```yaml
---
layout: cover
---
```

Selection priority:
1. Frontmatter `layout` field
2. Slide-index defaults (slide 0 → `cover`, others → `default`)
3. Theme defaults

Layouts are Vue components that wrap slide content.

## Build System

- **Compiler**: `tsdown` for all packages
- **Output**: TypeScript → `.mjs` + `.d.mts` files
- **Workspace aliases**: `tsconfig.json` has path aliases for packages (enables imports without build)
- **Watch mode**: `pnpm dev` watches and rebuilds on changes
- **Parallel builds**: `pnpm build` uses `pnpm -r --parallel`

### Package Build Scripts

Most packages use:
```json
{
  "build": "tsdown",
  "dev": "nr build --watch"
}
```

The CLI package specifically builds two entry points:
```json
{
  "build": "tsdown node/index.ts node/cli.ts"
}
```

## Client-Side Architecture

The Vue 3 client (`@slidev/client`) uses:

- **Main entry** (`main.ts`): Bootstraps app, injects context
- **Context injection**: `$slidev` global via Vue `provide/inject`
- **Composition API**: Composables for navigation, drawings, dark mode, etc.
- **Router**: Vue Router for slide navigation
- **Layouts**: Dynamic component wrapping based on frontmatter

## Development Workflow

### Working on Core Features

1. Make changes in relevant package (`packages/slidev/`, `packages/parser/`, etc.)
2. Run `pnpm dev` to watch and rebuild
3. Run `pnpm demo:dev` in another terminal to test with live demo
4. The dev server auto-restarts when builds complete

### Working on Parser

1. Edit files in `packages/parser/src/`
2. Tests are in `test/parser.test.ts` (root level)
3. Run `pnpm test parser` to test parser specifically

### Working on Client/Frontend

1. Edit files in `packages/client/`
2. Run `pnpm dev` to watch builds
3. Run demo to see changes live
4. Client uses HMR for fast feedback

### Working on VSCode Extension

1. Edit files in `packages/vscode/`
2. Run `pnpm vscode:dev` to launch extension development
3. Uses Volar language server

## Important Patterns

### Catalog System

The monorepo uses pnpm catalogs in `pnpm-workspace.yaml` to centralize dependency versions:
- `catalog:dev` - Development dependencies
- `catalog:prod` - Production dependencies
- `catalog:frontend` - Frontend libraries
- `catalog:types` - TypeScript type definitions
- `catalog:icons`, `catalog:monaco`, `catalog:themes`, `catalog:vscode` - Specialized groups

Reference in package.json as: `"vite": "catalog:prod"`

### Workspace Dependencies

Packages reference each other via `workspace:*`:
```json
{
  "dependencies": {
    "@slidev/parser": "workspace:*"
  }
}
```

### Request Path Conventions

- Virtual modules: `/@slidev/...`
- Slide modules: `__slidev_N.md` where N is slide index
- HTTP endpoints: `/slidev/...` for API calls

### Feature Detection

The parser detects which features are used:
- **KaTeX**: LaTeX math equations
- **Monaco**: Code editor
- **Mermaid**: Diagrams

This enables conditional loading of heavy dependencies.

## Testing

- **Framework**: Vitest
- **Test location**: `/test` directory at root
- **Test commands**: `pnpm test`
- **E2E tests**: Cypress (`/cypress` directory)

Test files:
- `test/parser.test.ts` - Parser tests
- `test/transform*.test.ts` - Markdown transformation tests
- `test/utils.test.ts` - Utility function tests

### Running Specific Tests

```bash
# Single test file
pnpm test parser

# Watch mode
pnpm test --watch
```

## Code Style

- **ESLint**: `@antfu/eslint-config`
- **Auto-formatting**: Git hooks with `lint-staged`
- Pre-commit hooks automatically format code
- Run `pnpm lint:fix` to manually format

## Common Modifications

### Adding a New Vite Plugin

1. Create plugin in `packages/slidev/node/plugins/`
2. Import and add to plugin array in `packages/slidev/node/plugins/preset.ts`
3. Follow existing plugin patterns (use `enforce` for ordering)

### Adding Parser Features

1. Modify parser logic in `packages/parser/src/core.ts` or `fs.ts`
2. Update types in `packages/types/src/`
3. Add tests in `test/parser.test.ts`
4. Update consumer code in CLI if needed

### Adding Client Features

1. Add composables to `packages/client/composables/`
2. Add components to `packages/client/components/`
3. Update context if needed in `packages/client/env.ts`
4. Use Vue 3 Composition API patterns

### Creating Themes

Use the scaffolding tool:
```bash
npm init slidev-theme
```

Theme structure:
- `layouts/` - Layout components
- `components/` - Reusable components
- `styles/` - Global styles
- `setup/` - Setup scripts
- `package.json` with `slidev` field

## Troubleshooting

### Build Issues

- Clear `node_modules` and reinstall: `rm -rf node_modules pnpm-lock.yaml && pnpm install`
- Rebuild all packages: `pnpm build`
- Check Node version: Requires Node >=18

### Type Errors

- Run `pnpm typecheck` to see all type errors
- Ensure all packages are built: `pnpm build`
- VSCode may need restart after package changes

### Demo Not Updating

- Check that `pnpm dev` is running (watch mode)
- Restart demo: Stop `pnpm demo:dev` and restart
- Clear Vite cache: Delete `demo/starter/.slidev/` and `node_modules/.vite/`

### Parser Issues

- Check markdown syntax in test fixtures: `test/fixtures/markdown/`
- Enable debug mode: Set `DEBUG=slidev:*` environment variable
- Parser uses `gray-matter` for frontmatter parsing

## References

- Documentation: https://sli.dev
- GitHub Issues: https://github.com/slidevjs/slidev/issues
- Discord: https://chat.sli.dev
