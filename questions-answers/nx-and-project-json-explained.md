# Nx and project.json Explained

Date: 2026-05-22

## Question

What is the purpose of a project.json file? What does the tool 'nx' do?

## Context

- Working with existing Nx workspace
- Evaluating Nx for projects
- Learning modern monorepo tooling
- Familiar with npm, not familiar with monorepos or task runners

## Answer Summary

### The Monorepo Problem

When you have multiple related projects in one repository:

```
company-workspace/
├── apps/
│   ├── web-app/
│   ├── mobile-app/
│   └── admin-dashboard/
└── libs/
    ├── shared-ui/
    ├── api-client/
    └── utils/
```

**Challenges:**
1. How to run commands across multiple projects?
2. What order to build things (dependencies)?
3. Avoiding unnecessary rebuilds when nothing changed
4. Managing how projects depend on each other

### What Is Nx?

**Nx is a build system and toolkit for managing monorepos.** Think of it as an intelligent orchestrator that sits above npm.

**Core Capabilities:**
1. **Task Running** - Like npm scripts, but across multiple projects
2. **Dependency Graph** - Understands relationships between projects
3. **Caching** - Never rebuild something that hasn't changed
4. **Task Orchestration** - Runs tasks in the right order, in parallel when possible

**Mental Model:**
Nx is like a project manager that:
- Knows which projects depend on others (by analyzing imports)
- Builds dependencies first, in parallel when possible
- Caches results and skips rebuilds when nothing changed
- Dramatically speeds up builds (5 minutes → 30 seconds)

### What Is project.json?

**project.json is Nx's version of package.json scripts, but with superpowers.**

**Comparison:**

**npm (package.json):**
```json
{
  "scripts": {
    "build": "tsc",
    "test": "jest"
  }
}
```

**Nx (project.json):**
```json
{
  "name": "web-app",
  "targets": {
    "build": {
      "executor": "@nx/js:tsc",
      "outputs": ["dist/apps/web-app"],
      "options": {
        "tsConfig": "apps/web-app/tsconfig.json"
      }
    }
  }
}
```

**Key Differences:**
1. **Targets** instead of scripts (you're targeting a specific task)
2. **Executors** instead of commands (pre-built, configurable task runners)
3. **Outputs** declaration (tells Nx what files are produced for caching)
4. **Implicit dependencies** (Nx analyzes imports to build dependency graph)

### How the Dependency Graph Works

**Nx analyzes your code automatically:**

In `apps/web-app/src/main.ts`:
```typescript
import { Button } from '@company/shared-ui';
import { ApiClient } from '@company/api-client';
```

**Nx sees this and builds a graph:**
```
    shared-ui ────┐
                  ├──→ web-app
    api-client ───┘
```

**When you run `nx build web-app`:**
1. Nx checks the graph
2. Realizes it needs to build dependencies first
3. Builds `shared-ui` and `api-client` in parallel (they're independent)
4. Then builds `web-app`

**You don't manage this manually - Nx figures it out from your imports!**

### The Caching System

**How it saves massive time:**

**First build:**
```bash
$ nx build web-app
```
1. Computes hash of inputs (source files, config, dependencies)
2. Runs the build
3. Stores output in cache (`dist/apps/web-app`)
4. Associates hash with cached output

**Second build (nothing changed):**
```bash
$ nx build web-app
```
1. Computes hash again
2. Hash identical (nothing changed!)
3. **Restores from cache**
4. Done in milliseconds instead of minutes!

**Incremental change:**
```bash
# You edit one file in shared-ui
$ nx build web-app
```
1. `shared-ui` changed → rebuild (can't cache)
2. `web-app` depends on changed `shared-ui` → rebuild
3. `api-client` unchanged → use cache ✓

### project.json Anatomy

```json
{
  "name": "web-app",                    // Project identifier
  "sourceRoot": "apps/web-app/src",     // Where source code lives
  "projectType": "application",         // "application" or "library"
  "targets": {                          // Tasks you can run
    "build": {
      "executor": "@nx/webpack:webpack", // Plugin that runs the task
      "outputs": ["{options.outputPath}"], // What files are created (for caching)
      "options": {                       // Configuration for the executor
        "outputPath": "dist/apps/web-app",
        "main": "apps/web-app/src/main.ts"
      },
      "configurations": {                // Different ways to run (prod vs dev)
        "production": {
          "optimization": true,
          "sourceMap": false
        }
      }
    },
    "test": {
      "executor": "@nx/jest:jest",
      "outputs": ["{workspaceRoot}/coverage/apps/web-app"]
    }
  },
  "tags": ["type:app", "scope:web"]     // Organizational labels
}
```

### Running Tasks with Nx

```bash
# Run a target for one project
nx build web-app
nx test web-app

# Run with configuration
nx build web-app --configuration=production

# Run for ALL projects
nx run-many --target=build --all

# Run for AFFECTED projects (changed since main branch)
nx affected --target=test
```

### The "Affected" Command: Smart CI/CD

**Incredibly powerful for large codebases:**

```bash
# You changed one file in shared-ui
nx affected --target=build
```

**Nx:**
1. Analyzes git diff: "What changed?"
2. Consults dependency graph: "What depends on changed code?"
3. Only builds: `shared-ui`, `web-app`, `mobile-app`
4. Skips: `admin-dashboard` (doesn't use `shared-ui`)

**Result:** In CI/CD, test only what could be broken, not the entire codebase!

### Task Orchestration: Parallel Execution

```bash
nx run-many --target=test --all
```

**Without Nx (sequential):**
```
test shared-ui        → 30s
test api-client       → 30s
test utils            → 30s
test web-app          → 60s
test mobile-app       → 60s
test admin-dashboard  → 60s
Total: 270s (4.5 minutes)
```

**With Nx (parallel, respecting dependencies):**
```
Batch 1 (parallel - no dependencies):
  shared-ui, api-client, utils → 30s

Batch 2 (parallel - depends on batch 1):
  web-app, mobile-app, admin-dashboard → 60s

Total: 90s (1.5 minutes)
```

**Nx uses all CPU cores effectively!**

### Nx vs npm Workspaces

**npm workspaces provides:**
- Dependency management (one node_modules)
- Package linking

**Nx adds:**
- **Task orchestration** - Runs builds in right order
- **Dependency graph** - Understands relationships
- **Caching** - Dramatically speeds up builds
- **Affected command** - Only tests what changed
- **Parallel execution** - Uses all CPU cores
- **Code generation** - Scaffolds new projects
- **Plugins** - Pre-built executors for popular tools

**You can use npm workspaces + Nx together!**

### Real-World Workflow Example

**Morning: Pull latest code**
```bash
git pull origin main
nx affected --target=build
```
Builds only what changed. 30 seconds instead of 10 minutes.

**Working on feature**
```bash
# Make changes to shared-ui
nx build shared-ui
nx test shared-ui

# See what will be affected
nx affected:graph
# Shows: shared-ui → web-app, mobile-app
```

**Ready to commit**
```bash
nx affected --target=test
nx affected --target=lint
git commit -m "feat: update button component"
```

**CI/CD Pipeline**
```yaml
- name: Test affected projects
  run: nx affected --target=test --base=origin/main
```
Only tests projects affected by PR. Speeds up CI dramatically.

### When to Use Nx

**Good fit:**
- Multiple related applications (web, mobile, admin)
- Shared libraries across apps
- Need fast CI/CD (affected tests)
- Large teams with many developers
- Complex build requirements

**Not necessary:**
- Single application
- Small projects
- Simple build needs (just `tsc` or `npm run build`)

**Bottom line:** Nx adds complexity, but if you have multiple projects that share code, the benefits are huge.

---

## Key Insights

1. **Nx = intelligent build orchestrator** for monorepos
2. **project.json = task definition** with executors, outputs, and dependencies
3. **Dependency graph** built automatically by analyzing imports
4. **Caching** based on input hashes - massive time savings
5. **Affected commands** only build/test what changed
6. **Parallel execution** uses all CPU cores, respects dependencies
7. **Nx sits above npm** - adds orchestration, caching, and intelligence

---

## Full Conversation

# Question Log
Started: 2026-05-22

## Phase 1: Log Start
✓ Logging initiated

## Phase 2: Question
User: "i wonder what the purpose for a project.json file is? i also would like to understand what the tool 'nx' does?"

## Phase 3: Clarification

Claude: "Are you:
1. Working with an existing Nx workspace and trying to understand what you're seeing?
2. Evaluating whether to use Nx for a project?
3. Just curious about how modern monorepo tooling works conceptually?

Do you have experience with monorepos, build tools like npm scripts, webpack, or task runners?"

User: "yes for 1, 2 and 3. i dont have experience with monorepos but im familiar with npm. not familiar with webpack and task runners."

## Phase 4: Answer

Claude provided comprehensive explanation covering:

**Monorepo Problem:**
- Multiple related projects in one repository (apps + shared libraries)
- Challenges: running commands, build order, avoiding unnecessary rebuilds, managing dependencies

**What Nx Is:**
- Build system and toolkit for managing monorepos
- Core capabilities: task running, dependency graph, caching, task orchestration
- Mental model: Nx as intelligent project manager that understands relationships and optimizes builds

**What project.json Is:**
- Nx's version of package.json scripts with superpowers
- Defines "targets" (tasks like build/test/lint) instead of "scripts"
- Uses "executors" (pre-built task runners) instead of shell commands
- Declares "outputs" for intelligent caching
- Enables Nx to understand project structure and dependencies

**Dependency Graph:**
- Nx analyzes import statements to build relationship map
- Automatically determines build order
- Enables parallel execution of independent tasks
- Example: web-app imports from shared-ui → Nx knows to build shared-ui first

**Caching System:**
- Computes hash of inputs (files, config, dependencies)
- Stores outputs in cache
- Reuses cached results when inputs haven't changed
- Dramatically speeds up rebuilds (minutes → milliseconds)

**project.json Anatomy:**
- name: project identifier
- projectType: "application" or "library"
- targets: tasks with executor, outputs, options, configurations
- tags: organizational labels

**Running Tasks:**
- Basic: `nx build web-app`
- Run many: `nx run-many --target=build --all`
- Affected: `nx affected --target=test` (only tests changed code and dependencies)

**Task Orchestration:**
- Parallel execution using all CPU cores
- Respects dependency order
- Example: 270s sequential → 90s parallel

**Nx vs npm workspaces:**
- npm workspaces: only dependency management
- Nx adds: orchestration, caching, affected commands, parallel execution, code generation, plugins

**When to use Nx:**
- Multiple related applications
- Shared libraries
- Need fast CI/CD
- Large teams
- Not necessary for single small applications

Full examples and real-world workflow provided.

## Phase 5: Confirmation

Claude: "Does this explanation make sense? Do you understand what Nx does, how project.json configures projects, and how the caching and dependency graph work?"

User: "yes"

---

Completed: 2026-05-22
