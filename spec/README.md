# Todos — Specification

> An installable, offline-first PWA that shows a list of todos with due
> dates. Spec-first: an implementer reading this spec should be able to
> ship a working app in one shot.

---

## Deliverable — REQUIRED

**The deliverable is a scannable QR code printed by `npm run serve:phone`
that resolves to an installable PWA on the implementer's phone.** That QR
is the finish line. Everything else in this spec — types, tables,
components, smoke test — exists to protect it.

A run that passes typecheck, build, and smoke but never prints the QR
**has not shipped**. A run that prints a QR pointing at a tunnel the
phone cannot install from **has not shipped**. The QR must print, and
its target must be installable. Both are required.

This requirement is restated under § "Success Criteria" (#10),
§ "Quality Bar", § "MVP Cut", § "Implementation Plan", and
§ "Install flow — `serve:phone`".
If you find this repetitive, that is the intent.

The **main agent** — the interactive coding agent the user opened in
the repo (Claude Code or equivalent) — owns this final step. Subagents
do not run `serve:phone`. **The main agent is responsible for making
the complete QR visible to the user: it must render the entire QR
block (every row, untruncated) and the `*.trycloudflare.com` URL
beneath it in its own reply — not merely let `serve:phone`'s output
scroll past in a tool buffer.** A truncated, summarized, or "see
above" QR **has not shipped**. The process must also stay alive (tunnel
up) until the user stops it.

One invariant, two runtimes. The invariant: the full QR ends up in
front of the user, and the tunnel stays up until they install.
- **At a human terminal:** run `serve:phone` in the **foreground**,
  attached, with stdout not redirected away, so the QR renders
  directly to the screen. Stay attached until Ctrl-C.
- **In an agent / automation harness (no shared terminal):** the main
  agent **captures** `serve:phone`'s output, waits for the QR and URL
  to appear, **reprints the entire QR block verbatim** in its reply,
  and leaves the process running so the tunnel stays up. Capturing the
  output is **required** here, not forbidden — it is the only way the
  QR reaches the user.

What defeats the deliverable is the QR never becoming fully visible to
the user, a truncated QR, or the tunnel being torn down before the
user has installed.

## Core User Story

The user opens the app and sees their todo items, each showing a title
and a due date. They tap **Add**, enter a title and a date, and the new
item appears in the list, sorted by due date. They tick items off as
they finish them. They can delete items they no longer want. Data
persists locally, survives reloads, and works offline.

That is the entire user-facing surface.

## Success Criteria

A build is considered done when every item passes. Items 1–9 are gates
that protect item 10. **Item 10 — the QR — is the deliverable.** If
1–9 pass and 10 fails, the run failed. If 10 passes and 1–9 are
unverified, you skipped the gates.

1. `npm install`, `npm run build`, and `npm run preview` complete with
   no TypeScript or bundler errors.
2. On first open, the app shows an empty state and an **Add** affordance.
3. Tapping Add reveals an inline form (title input + date input + Save).
   Submitting with a non-empty title and a valid date creates the todo.
4. The list shows all todos sorted: open items first by earliest due
   date; completed items below by most-recently-completed first.
   Completed items render with strikethrough.
5. Each row's checkbox toggles the item between open and completed.
6. Each row's delete control removes the todo after a confirmation.
7. Todos persist across reloads. The app loads with the network
   disconnected after the first visit (service worker).
8. The app is installable as a PWA: valid manifest, registered service
   worker, **PNG** icons at 192×192 and 512×512 served from
   `/icons/icon-192.png` and `/icons/icon-512.png`. Chrome's
   installability check rejects SVG-only manifests — the QR will
   scan and the app will load, but the "Install app" affordance
   never appears. PNG is mandatory; SVG is fine as a supplementary
   `<link rel="icon">` and apple-touch-icon.
9. `node scripts/smoke.mjs` — which frees its own dedicated port
   42730 (never 41730) and boots and tears down its own preview server
   there — adds, completes, and deletes a todo. Exit code 0.
10. `npm run serve:phone` produces exactly one terminal QR and one
    `https://*.trycloudflare.com` URL underneath it. The script
    always runs `npm run build` first (never serve stale), then
    verifies as a **pre-flight** that the root document is served
    200 with a `<link rel="manifest">`, that the manifest is valid
    JSON with at least one 192 and one 512 icon entry, and that
    every icon URL returns 200. If any pre-flight check fails, no QR
    is printed and the script exits non-zero with a clear one-line
    message. **The QR appearing is the install-flow gate.**

## Non-Goals (everything not above is out of scope)

Cut by name, so an implementer can't accidentally rebuild them:

- Multiple lists. There is one list.
- Tags, priorities, descriptions, attachments, sub-tasks, recurrence,
  drag-to-reorder, sharing, multi-user.
- Notifications, reminders, push, badges.
- Theme toggle. The app follows the system colour scheme automatically.
- Settings page. There is no settings page.
- Search, filtering, day grouping, "Today / Upcoming / Inbox" views.
- Editing a todo's title or date. To correct a mistake, delete and
  re-add.
- Server sync, accounts, auth.
- Full test suite. `scripts/smoke.mjs` (per-screen behaviour) is the
  verification oracle for in-page UX. The install-flow check is built
  into `scripts/serve-phone.mjs` as a pre-flight; if the QR prints,
  the install path is good. Vitest may be added later but is not
  required for delivery.

## Tech Stack

| Concern | Choice | Why |
|---|---|---|
| Build | Vite 5 | Fast, first-class PWA plugin |
| UI | React 18 + TypeScript 5 (strict) | Standard, low-friction |
| Styling | Tailwind 3 | Utility-first; no bespoke CSS |
| Storage | Dexie 4 | Typed IndexedDB wrapper |
| PWA | vite-plugin-pwa (Workbox) | SW + manifest generation |
| IDs | ulid | Sortable, client-generated |
| Smoke | puppeteer-core | Against the system Chrome |

Explicitly **not** in the stack: TanStack Query, Zustand, React Router,
Zod, react-hook-form, lucide-react, date-fns, workbox-window, @dnd-kit,
Playwright. Each was un-justified by the user story.

## Architecture

Three layers, dependencies pointing inward.

```
   ┌──────────────────────────────────────────┐
   │  ui/      React app. One screen.         │
   └──────────────────┬───────────────────────┘
                      │ depends on
                      ▼
   ┌──────────────────────────────────────────┐
   │  data/    Dexie schema + repository.     │
   │           Returns/accepts domain types.  │
   └──────────────────┬───────────────────────┘
                      │ depends on
                      ▼
   ┌──────────────────────────────────────────┐
   │  domain/  Types, validation, IDs.        │
   │           Pure TypeScript.               │
   └──────────────────────────────────────────┘
```

Rules:

- `domain/` is pure. Imports only `ulid`. No DOM, no React, no Dexie,
  no `Date.now()` (callers pass `now: number` if needed).
- `data/` imports `@/domain` and `dexie`. Exposes a singleton
  `todoRepository` from its barrel.
- `ui/` imports `@/data` and `@/domain` types only. No direct Dexie
  access; all persistence goes through the repository.

There is no `application/` layer. There are no use-case services. There
is no `platform/` layer because the app uses no browser capabilities
beyond what React itself needs. The architectural weight class is
matched to the product's actual size.

## Quality Bar

- TypeScript `strict: true`, plus `noUnusedLocals` and
  `noUnusedParameters`.
- All form controls have an associated label or `aria-label`.
- Tap targets are at least 44×44 px (`min-h-11`).
- `:focus-visible` ring on every focusable control.
- Mobile-first: layout works at 360 px viewport width.
- Safe-area padding for notched devices
  (`env(safe-area-inset-top/bottom)`).
- Light/dark via `prefers-color-scheme` CSS only — no JS toggle.
- Bundle: under 250 KB gzipped JS for `dist/`.
- Smoke test passes against `npm run preview`.
- **REQUIRED:** `npm run serve:phone` prints a QR. Pre-flight gates
  the install path; see § "Install flow — `serve:phone`". **No QR, no
  ship.**

## Project Layout

```
todos/
  spec/
    README.md                      (this file — the whole spec)

  public/
    icons/
      make-icons.mjs          # pure-Node PNG writer; emits the two PNGs
      icon-192.png            # produced by make-icons.mjs (solid dark square)
      icon-512.png            # produced by make-icons.mjs

  src/
    main.tsx
    App.tsx

    domain/
      types.ts              # Todo, TodoStatus, TodoInput
      ids.ts                # TodoId (branded), newTodoId, parseTodoId
      rules.ts              # validateTodoInput
      errors.ts             # DomainError, ValidationError, NotFoundError
      index.ts              # barrel

    data/
      db.ts                 # Dexie schema (one table)
      todo-repository.ts    # singleton repo
      index.ts              # barrel: todoRepository + re-exported types

    ui/
      todo-app.tsx          # the whole screen
      todo-form.tsx         # collapsed "Add" → inline form
      todo-row.tsx          # one row with checkbox + title + date + delete
      use-todos.ts          # list + create + setStatus + delete
      styles.css            # Tailwind directives + CSS variables

  scripts/
    free-port.mjs           # freePort(port) helper + `npm run clean` CLI
    serve-phone.mjs         # build + preview + pre-flight + cloudflared + QR
    smoke.mjs               # headless smoke (puppeteer-core)

  index.html
  package.json
  vite.config.ts
  tailwind.config.ts
  postcss.config.cjs
  tsconfig.json
  README.md
```

## Install Flow

The PWA reaches the phone via one command on the laptop and one scan
on the phone.

### Laptop side

```powershell
npm install
npm run serve:phone
```

`scripts/serve-phone.mjs` (described fully in
§ "Install flow — `serve:phone`"):

1. Always runs `npm run build` first. The user may have edited the
   spec since the last run — never serve a stale `dist/`.
2. Boots `vite preview` on port 41730.
3. Polls until the local server responds.
4. **Pre-flight:** fetches `/`, `/manifest.webmanifest`, and every
   icon URL from the manifest. All must return 200, the manifest must
   parse, and it must contain at least one 192 and one 512 icon. If
   any pre-flight check fails, the script aborts with a one-line
   message and never opens the tunnel — there is no broken QR to
   scan.
5. Boots `cloudflared tunnel --url http://localhost:41730`.
6. Watches cloudflared's stdout for a `https://*.trycloudflare.com`
   URL. Captures the first one.
7. Renders that URL as a terminal QR via `qrcode-terminal`, prints
   the URL on its own line beneath it, and prints once per run (later
   matches in cloudflared output are suppressed).
8. Forwards both child logs; on Ctrl+C, terminates both children.

### Phone side — what the user does beyond scanning

Scanning the QR is not enough on its own. Browsers and operating
systems gate PWA install behind a few taps. **The spec cannot remove
these — they are imposed by iOS / Android, not the app.** What the
user does:

| Step | iOS (iPhone / iPad) | Android |
|---|---|---|
| 1. Scan QR | Camera app. Tap the URL banner. | Camera app (Android 8+). Tap the URL. |
| 2. Open in the right browser | Must open in **Safari**. Chrome on iOS cannot install PWAs. If the link opens in another app, copy it into Safari. | Must open in **Chrome**. |
| 3. Wait for load | First load needs the tunnel; subsequent launches are offline. | Same. |
| 4. Install | **Share (□↑) → Add to Home Screen → Add.** No automatic install banner on iOS. | "Install app" banner usually appears; tap it. If not: menu (⋮) → *Install app*. |
| 5. Launch | Tap the new home-screen icon. | Tap the new home-screen icon. |

Steps 2 and 4 are the only ones the spec cannot collapse. iOS
specifically requires the user to know about the Share → Add to Home
Screen path — there is no system prompt.

The laptop terminal must stay up until the install completes on the
phone. Operator-side caveats (single-use tunnel URLs, corporate Wi-Fi
blocking trycloudflare, etc.) live in the root `README.md`
§ "Troubleshooting" — they are user-facing, not spec material.

## Implementation Plan

The work partitions into **six independent modules**. Each module is
one agent. The six agents **parallelize** — they run concurrently,
have no execution order between them, and never wait on each other.
Within each module, the agent **fans out** its file writes: every
file the module owns is emitted in a single message with parallel
Write calls. There is no internal ordering inside a module either.

**This is a spec, not code.** TypeScript only resolves at the verify
step at the end of the run. Until then, every module is just files on
disk in a separate folder. Two agents writing files in different
folders cannot conflict, so they fan out concurrently.

The **main agent** does not write source files. It is the interactive
coding agent the user opened in the repo (Claude Code or equivalent).
Its job: read the spec, spawn the six module agents, kick `npm install`
in the background, run the final verification chain, and run
`npm run serve:phone` so the QR appears in its own terminal — in
contact with the user. Subagents do not run `serve:phone`.

### The six modules

| Agent | Owns | Files | Reads | Done when |
|---|---|---|---|---|
| **domain** | `src/domain/` | `errors.ts`, `ids.ts`, `types.ts`, `rules.ts`, `index.ts` (5) | § "Domain Layer" | All 5 exist. Barrel re-exports the other four. Zero DOM / React / Dexie imports. |
| **data** | `src/data/` | `db.ts`, `todo-repository.ts`, `index.ts` (3) | § "Data Layer" + domain types | All 3 exist. Repository exposes `list` / `create` / `setStatus` / `delete`. No Dexie types leak through the barrel. |
| **ui** | `src/App.tsx`, `src/main.tsx`, `src/ui/` | `App.tsx`, `main.tsx`, `ui/styles.css`, `ui/use-todos.ts`, `ui/use-install-prompt.ts`, `ui/todo-row.tsx`, `ui/todo-form.tsx`, `ui/todo-app.tsx` (8) | § "Frontend" (Selector Contract is binding) | All 8 exist. Selectors, aria-labels, and DOM shape match § "Selector Contract" **verbatim** — smoke asserts literal strings. |
| **scripts** | `scripts/` | `free-port.mjs`, `serve-phone.mjs`, `smoke.mjs` (3) | § "Infrastructure", § "Selector Contract" | All three exist. `free-port.mjs` exports `freePort(port)` (bind-probe → tree-kill any listener) and also runs as a CLI; both `serve-phone` and `smoke` import it. `serve-phone.mjs` frees **port 41730** first, then build + pre-flight + cloudflared + exactly one QR print. `smoke.mjs` runs on its **own dedicated port 42730 (never 41730)** — frees it, boots and tears down its own preview — and uses the native `HTMLInputElement` value setter for the React date input (`Object.getOwnPropertyDescriptor(proto, 'value').set` — direct `.value =` is swallowed by React). |
| **icons** | `public/icons/` | `make-icons.mjs` (writes itself, then runs to produce `icon-192.png` + `icon-512.png`) | § "Icons" | `make-icons.mjs` exists and has been run. `icon-192.png` and `icon-512.png` exist at the right path. Both PNGs decode at the exact pixel dimensions. The agent writes the helper from the spec, runs it once, and is done. |
| **configs** | repo root | `package.json`, `tsconfig.json`, `vite.config.ts`, `tailwind.config.ts`, `postcss.config.cjs`, `index.html` (6) | § "Tech Stack", § "Infrastructure" | All 6 exist. `package.json` dependency list matches the infrastructure spec exactly. |

**Total: 26 files across 6 parallel agents.** (The icons module
writes 1 helper plus 2 generated PNGs; only the helper is an agent
Write.)

### Main-agent orchestration

1. **Read the spec.** The entire spec is this one file
   (`spec/README.md`). The Implementation Plan table above is the
   complete handoff context the main agent needs for orchestration.
   Hand each module agent the section named in its "Reads" column —
   an agent needs only its own section, not the whole spec. The point
   is to keep each agent's context tight at spawn time.
2. **Spawn the modules.** In one message, spawn all 6 module agents
   with parallel Agent calls. Hand each agent its row from the table
   above.
3. **Install in the background.** The moment `package.json` exists on
   disk, kick `npm install` as a background command. It takes 30–90 s
   and runs concurrently with the module agents. If the harness can't
   watch for the file landing, kick `npm install` immediately after
   step 2 — it waits for `package.json` to appear on its own.
4. **Wait** for all 6 module agents and `npm install` to return.
5. **Verify — concurrent.** In one message, run `npm run typecheck`
   and `npm run build`. Both must pass. This is where the modules
   connect: any cross-module type mismatch surfaces here, not at
   write time.
6. **Smoke.** Run `npm run smoke`; assert exit 0. The smoke script
   runs on its **own dedicated port (42730), never 41730**: it frees
   42730 and boots **and tears down** its own preview there
   (free → preview → puppeteer → tree-kill), so the orchestrator does
   **not** start a preview for it. There is no `npm run preview &`
   step — that stray ampersand is what orphaned a port and blocked the
   next run. Because smoke never touches 41730, the smoke gate and the
   final serve step can never contend for the deliverable port.
7. **Ship.** Run `npm run serve:phone`. Its stdout carries the QR. The
   main agent must get the **complete** QR in front of the user and
   keep the tunnel up — see § "Deliverable" for the invariant and the
   two runtime modes:
   - **At a human terminal:** run it in the **foreground**, attached,
     stdout not redirected, and stay attached until Ctrl-C.
   - **In an agent harness (no shared TTY):** start it, watch its
     output until the `*.trycloudflare.com` URL and QR block appear,
     then **reprint the entire QR block (all rows) and the URL verbatim
     in your reply**, and leave the process running so the tunnel stays
     up.

   Either way: never tear the process down before the user has
   installed, and never show a partial or truncated QR. **Do not
   declare success until the full QR is on screen in your reply.**

## MVP Cut

The "smallest subset that still ships the user story":

Required: form, list, repository, persistence, PWA install path,
**terminal QR**. The QR is the user-visible finish line of an install
run; it is part of the MVP.

Could be dropped if running out of time and still call it shipped:

- Delete control on rows (users could complete instead).

Everything else in this spec is part of the MVP cut.

## Decisions and Future Work

- **Decision (v1):** Single-screen, single-list. No multi-list types.
- **Decision (v1):** No edit. Misspellings are fixed by delete + re-add.
- **Decision (v1):** Completed items stay in the same list with
  strikethrough, not under a separate "Done" page.
- **Decision (v1):** No update toast. `vite-plugin-pwa` registers in
  `autoUpdate` mode and updates silently on next reload.
- **Decision (v1):** A small **Install** button appears in the header
  on Android Chrome when `beforeinstallprompt` fires. Reason: the
  final artifact is a scannable QR whose destination is actually
  installable. Chrome's own install affordance (mini-infobar, menu
  entry) is unreliable — engagement heuristics, dismissal cooldowns,
  and Chrome-variant browsers can all hide it. An in-app button that
  calls `prompt()` from a user gesture is the documented reliable
  path. The button is invisible on iOS Safari (no event fires there)
  and after install completes.
- **Future:** Inline edit on tap.
- **Future:** "Clear completed" bulk action.
- **Future:** Search / filtering.

## Editing this spec

Keep a code block canonical if removing it would let an implementer
satisfy the contract with code that ships a different user
experience. Otherwise prefer prose, a table, or a partial snippet.
That rule decides what stays and what trims.

---

## Domain Layer

> Pure types and validation for the Todo entity. Depends only on `ulid`.

This is the smallest layer the app needs and the only one with rules
that change least over time.

This is the **domain** module. Orchestration (when to run, how
to spawn, when to verify) lives in § "Implementation
Plan" — this section describes only what to build.

### Files

```
src/domain/
  types.ts
  ids.ts
  rules.ts
  errors.ts
  index.ts
```

### Dependencies

- `ulid` — for ID generation.

Nothing else. In particular, no DOM, no React, no Dexie, no `date-fns`,
no `zod`, no calls to `Date.now()`. If a function needs the current
time, it takes `now: number` as a parameter so tests are deterministic.

### Types

```ts
// src/domain/types.ts

import type { TodoId } from './ids'

export type TodoStatus = 'open' | 'completed'

export interface Todo {
  readonly id: TodoId
  title: string
  /** Epoch ms at local midnight on the due day. */
  dueAt: number
  status: TodoStatus
  readonly createdAt: number
  completedAt?: number
}

export interface TodoInput {
  title: string
  dueAt: number
}
```

The entity has six fields. Each is required by the user story:

| Field | Why |
|---|---|
| `id` | Stable reference for toggle/delete operations |
| `title` | The user typed it |
| `dueAt` | The user picked a date |
| `status` | Checkbox state |
| `createdAt` | Stable tiebreak in sort, useful for debugging |
| `completedAt` | Sort key for the completed section |

No `description`, `priority`, `reminderAt`, `tagIds`, `position`,
`listId`. They have no place in the user story.

### IDs

```ts
// src/domain/ids.ts

import { ulid } from 'ulid'
import { ValidationError } from './errors'

export type TodoId = string & { readonly __brand: 'TodoId' }

export const newTodoId = (): TodoId => ulid() as TodoId

const ULID_RE = /^[0-9A-HJKMNP-TV-Z]{26}$/

export function parseTodoId(value: string): TodoId {
  const normalized = value.trim().toUpperCase()
  if (!ULID_RE.test(normalized)) throw new ValidationError('Invalid todo id')
  return normalized as TodoId
}
```

The brand prevents arbitrary strings being passed where a `TodoId` is
expected. Zero runtime cost.

`parseTodoId` is the canonical parser for any external-string source
(storage rehydration, URL params, imports). Keep it even when no UI
code calls it directly — its absence would force `as TodoId` casts
that defeat the brand.

### Validation

```ts
// src/domain/rules.ts

import { ValidationError } from './errors'
import type { TodoInput } from './types'

export function validateTodoInput(input: TodoInput): void {
  const title = input.title.trim()
  if (title.length < 1) throw new ValidationError('Title is required')
  if (title.length > 200) throw new ValidationError('Title is too long')
  if (!Number.isFinite(input.dueAt) || input.dueAt <= 0) {
    throw new ValidationError('Due date is required')
  }
}
```

A single function is the rule surface. The data layer calls it before
inserting; the UI surfaces the thrown message to the user.

No Zod, because there is one input shape and one validator.

### Errors

```ts
// src/domain/errors.ts

export class DomainError extends Error {
  constructor(message: string, public readonly cause?: unknown) {
    super(message)
    this.name = this.constructor.name
  }
}

export class ValidationError extends DomainError {}
export class NotFoundError extends DomainError {}
```

Two subclasses cover the actual throw sites in this product
(`validateTodoInput` and the repository's missing-record paths).

### Barrel

```ts
// src/domain/index.ts
export * from './types'
export * from './ids'
export * from './rules'
export * from './errors'
```

### Boundary Rules

Forbidden in `domain/`:

- DOM types, `window`, `navigator`, `localStorage`, `Notification`,
  `Date.now()`.
- React imports.
- `dexie`, the `db` instance, or anything from `@/data`.
- `date-fns`, `zod`, or any other helper library.

If a domain function needs the current time, take it as a parameter.

### Tests

Optional but recommended. The rule surface is small enough to cover in
one file:

```ts
// tests/unit/rules.test.ts
import { describe, it, expect } from 'vitest'
import { validateTodoInput, ValidationError } from '@/domain'

describe('validateTodoInput', () => {
  it('rejects empty title', () => {
    expect(() => validateTodoInput({ title: '   ', dueAt: 1 })).toThrow(ValidationError)
  })
  it('rejects missing dueAt', () => {
    expect(() => validateTodoInput({ title: 'x', dueAt: 0 })).toThrow(ValidationError)
  })
  it('accepts a valid input', () => {
    expect(() => validateTodoInput({ title: 'x', dueAt: 1 })).not.toThrow()
  })
})
```

Vitest is not required for delivery (per § "Non-Goals")
but the file lives here once added.

---

## Data Layer

> One Dexie table. One repository. Returns and accepts only domain types.

This layer owns persistence and nothing else. The UI never sees Dexie.

This is the **data** module. Orchestration lives in
§ "Implementation Plan" — this section describes only
what to build.

### Files

```
src/data/
  db.ts
  todo-repository.ts
  index.ts
```

### Dependencies

- `dexie`
- `@/domain`

### Schema

```ts
// src/data/db.ts

import Dexie, { type Table } from 'dexie'
import type { Todo } from '@/domain'

export class TodosDB extends Dexie {
  todos!: Table<Todo, string>

  constructor() {
    super('todos-app')
    this.version(1).stores({
      todos: 'id, status, dueAt, [status+dueAt]',
    })
  }
}

export const db = new TodosDB()
```

Index rationale:

| Index | Used by |
|---|---|
| `id` | Primary key, lookups by id |
| `status` | Filtering open vs completed |
| `dueAt` | Sorting by due date |
| `[status+dueAt]` | The main list query |

No migrations. If the schema ever needs to change, a new `.version(2)`
block is added; the old block is never edited.

There is no seed step. The app starts empty.

### Repository

```ts
// src/data/todo-repository.ts

import {
  newTodoId,
  NotFoundError,
  validateTodoInput,
  type Todo,
  type TodoId,
  type TodoInput,
  type TodoStatus,
} from '@/domain'
import { db } from './db'

export interface TodoRepository {
  list(): Promise<Todo[]>
  create(input: TodoInput): Promise<Todo>
  setStatus(id: TodoId, status: TodoStatus): Promise<Todo>
  delete(id: TodoId): Promise<void>
}

class TodoRepositoryImpl implements TodoRepository {
  async list(): Promise<Todo[]> {
    const rows = await db.todos.toArray()
    rows.sort((a, b) => {
      if (a.status !== b.status) return a.status === 'open' ? -1 : 1
      if (a.status === 'open') return a.dueAt - b.dueAt
      return (b.completedAt ?? 0) - (a.completedAt ?? 0)
    })
    return rows
  }

  async create(input: TodoInput): Promise<Todo> {
    validateTodoInput(input)
    const now = Date.now()
    const todo: Todo = {
      id: newTodoId(),
      title: input.title.trim(),
      dueAt: input.dueAt,
      status: 'open',
      createdAt: now,
    }
    await db.todos.add(todo)
    return todo
  }

  async setStatus(id: TodoId, status: TodoStatus): Promise<Todo> {
    const existing = await db.todos.get(id)
    if (!existing) throw new NotFoundError(`Todo ${id} not found`)
    const next: Todo =
      status === 'completed'
        ? { ...existing, status, completedAt: Date.now() }
        : { id: existing.id, title: existing.title, dueAt: existing.dueAt,
            status: 'open', createdAt: existing.createdAt }
    await db.todos.put(next)
    return next
  }

  async delete(id: TodoId): Promise<void> {
    const existing = await db.todos.get(id)
    if (!existing) throw new NotFoundError(`Todo ${id} not found`)
    await db.todos.delete(id)
  }
}

export const todoRepository: TodoRepository = new TodoRepositoryImpl()
```

Four methods. Each maps 1:1 to a user-story action (list, add,
toggle, delete). `setStatus` is one method instead of two
(`complete`/`uncomplete`) because the UI only needs to mirror the
checkbox state.

The repository is a singleton. The UI imports the instance. There is
no DI container; a single entity and a single consumer do not warrant
one.

### Barrel

```ts
// src/data/index.ts
export { todoRepository, type TodoRepository } from './todo-repository'
export type { Todo, TodoId, TodoStatus, TodoInput } from '@/domain'
```

Re-exporting the domain types means the UI imports everything it needs
from `@/data` and never sees Dexie.

### Boundary Rules

Allowed:
- `dexie`, `@/domain`.

Forbidden:
- React, DOM globals, `window`, `navigator`.
- Any application-layer abstractions (`UnitOfWork`, ports, container).
  Those do not exist; do not invent them.

### Errors

- `validateTodoInput` may throw `ValidationError` during `create`.
- `setStatus` and `delete` throw `NotFoundError` if the row is missing.

The UI catches these at the call site (the hook in `ui/use-todos.ts`).
Dexie's own errors propagate uncaught; the user story has no
specifically-translated failure modes for them in v1.

### Tests

Optional. If added: `tests/integration/todo-repository.test.ts` using
`fake-indexeddb/auto`, exercising `list`, `create`, `setStatus`,
`delete`.

Not required for delivery — the smoke test (`scripts/smoke.mjs`)
exercises the repository end-to-end through the UI.

---

## Frontend

> One screen. One form. One list. Tailwind for styling. PWA shell
> auto-generated by `vite-plugin-pwa`.

This replaces v0.1's `pages.md`, `components.md`, and `pwa.md`. Routing,
multi-page navigation, theme toggle, sidebar, mobile nav, drag/drop,
forms library, and global state library are all out of scope per
§ "Non-Goals".

This is the **ui** module (it also owns `src/App.tsx` and
`src/main.tsx`, which are siblings of `src/ui/`). Orchestration lives
in § "Implementation Plan" — this section describes only
what to build. The **Selector Contract** below is binding: every
selector, aria-label, and DOM shape must match it verbatim because the
smoke test asserts literal strings.

**Why this module matters for the deliverable.** The QR that
`serve:phone` prints is only useful if the page it points at is an
installable PWA. The **PWA manifest**, **registered service worker**,
and **PNG icons** described below are what make the target installable
— those three are what Chrome's installability check actually reads.
The **Selector Contract** is what `scripts/smoke.mjs` asserts against
this module. The **install-prompt hook** adds the in-app Install
button (described below) on top of Chrome's own install affordance.

### Files

```
src/
  main.tsx
  App.tsx
  ui/
    todo-app.tsx          # the whole screen
    todo-form.tsx
    todo-row.tsx
    use-todos.ts
    styles.css
```

That is the whole frontend.

### Entry

```ts
// src/main.tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import { App } from './App'
import './ui/styles.css'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

```tsx
// src/App.tsx
import { TodoApp } from './ui/todo-app'

export function App() {
  return <TodoApp />
}
```

No router. No providers. No context. There is one screen.

### The screen

```
┌──────────────────────────────────────────┐
│  Todos                          [Install]│ ← Install shown only when
│                                          │   beforeinstallprompt has fired
│  [+ Add]                                 │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ □  Buy milk          May 18    [×] │  │
│  │ □  Pay rent          May 30    [×] │  │
│  │ ─────────────────────────────────  │  │
│  │ ☑  Old item          May 14    [×] │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

`TodoApp` renders, in order:
1. A header row with an `<h1>` "Todos" on the left and, when
   available, an inline `<button>` "Install" on the right.
2. `<TodoForm>` (collapsed Add button, or the inline form).
3. The sorted list of `<TodoRow>`s.
4. Empty-state text when the list is empty.

A subtle horizontal divider sits between the last open todo and the
first completed todo when both groups are present.

### Selector Contract

`scripts/smoke.mjs` is the verification oracle for the per-screen
behaviour in § "Success Criteria" #1–#9. It asserts
against the **exact** strings, attributes, and DOM shapes listed
below. The implementation must produce these literally; the smoke
test must read this section as the source of truth and refer to it
from its own header.

If you change a selector here, change the matching assertion in
`smoke.mjs` in the same commit. They are a paired contract.

| What | Selector / value | When |
|---|---|---|
| Empty state | The document body contains the literal text `Nothing to do.` | Whenever the todo list is empty |
| Add button (collapsed form) | `<button aria-label="Add todo">` | When `TodoForm` is collapsed |
| Title input | `<input aria-label="Title" type="text">` | When the form is expanded |
| Due-date input | `<input aria-label="Due date" type="date">` | When the form is expanded |
| Save button | `<button type="submit">` whose trimmed text content is exactly `Save` | When the form is expanded |
| Cancel button | `<button type="button">` whose trimmed text content is exactly `Cancel` | When the form is expanded |
| Validation error | `<p role="alert">` containing the thrown error's `message` | After a failed submit |
| Todo row | Rendered inside an `<li>` element (one `<li>` per todo) | Always |
| Todo title element | A `<span>` inside that `<li>`, whose `textContent` is **exactly** the todo title (no extra whitespace, no decorating characters) | Always |
| Strikethrough on completed title | The same title `<span>` gains the Tailwind class `line-through` (used literally — not aliased) | When the row's status is `completed` |
| Row checkbox | `<input type="checkbox" aria-label='Mark "<title>" as <verb>'>` — `<verb>` is `complete` when status is `open`, `open` when status is `completed`. The `aria-label` always starts with `Mark "<title>"` (note the quoted title). | Always; `checked` reflects status |
| Delete button | `<button aria-label='Delete "<title>"'>` (literal double-quotes around the title) | Always |
| Delete confirmation | `window.confirm('Delete this todo?')` is the prompt. Anything truthy from the dialog deletes; falsy keeps the row. | When the delete button is clicked |

Notes:

- The angle-brackets above (`<title>`, `<verb>`) are placeholders;
  the rendered `aria-label` contains the actual title and verb.
- `Mark "<title>"` and `Delete "<title>"` use **straight double
  quotes** (`"`), not typographic quotes. Smoke matches the literal
  ASCII character.
- All other DOM choices (wrapping `<div>`s, additional classes,
  spacing utilities) are at the implementer's discretion. The
  contract above is the only thing smoke asserts.

### Install button

Files:

```
src/ui/
  use-install-prompt.ts
```

The hook:

- Captures `beforeinstallprompt` on `window`, prevents the default
  mini-infobar (it's unreliable per the events-spec comments above),
  and exposes a `promptInstall()` function plus a boolean
  `canInstall`.
- Listens to `appinstalled` on `window`. Sets `canInstall` back to
  false so the button hides after install.
- Returns `{ canInstall: false }` on iOS Safari and other browsers
  that never fire the event. Nothing to render.

```ts
// src/ui/use-install-prompt.ts

import { useCallback, useEffect, useState } from 'react'

interface BeforeInstallPromptEvent extends Event {
  prompt: () => Promise<void>
  userChoice: Promise<{ outcome: 'accepted' | 'dismissed' }>
}

export function useInstallPrompt() {
  const [deferred, setDeferred] =
    useState<BeforeInstallPromptEvent | null>(null)

  useEffect(() => {
    const onPrompt = (e: Event) => {
      e.preventDefault()
      setDeferred(e as BeforeInstallPromptEvent)
    }
    const onInstalled = () => setDeferred(null)
    window.addEventListener('beforeinstallprompt', onPrompt)
    window.addEventListener('appinstalled', onInstalled)
    return () => {
      window.removeEventListener('beforeinstallprompt', onPrompt)
      window.removeEventListener('appinstalled', onInstalled)
    }
  }, [])

  const promptInstall = useCallback(async () => {
    if (!deferred) return
    await deferred.prompt()
    await deferred.userChoice
    setDeferred(null)
  }, [deferred])

  return { canInstall: deferred !== null, promptInstall }
}
```

In `todo-app.tsx`, render `<button aria-label="Install app">Install</button>`
inside the header row only when `canInstall` is true. `onClick`
calls `promptInstall()` (user gesture preserved). No selector
contract entry — the smoke test runs in headless Chrome which does
not fire `beforeinstallprompt`; this button is exercised on real
devices only.

### Hook

```ts
// src/ui/use-todos.ts

import { useCallback, useEffect, useState } from 'react'
import {
  todoRepository,
  type Todo,
  type TodoId,
  type TodoInput,
  type TodoStatus,
} from '@/data'

export function useTodos() {
  const [todos, setTodos] = useState<Todo[] | null>(null)
  const [error, setError] = useState<Error | null>(null)

  const reload = useCallback(async () => {
    try {
      setTodos(await todoRepository.list())
    } catch (err) {
      setError(err instanceof Error ? err : new Error(String(err)))
    }
  }, [])

  useEffect(() => { void reload() }, [reload])

  const create = useCallback(async (input: TodoInput) => {
    await todoRepository.create(input)
    await reload()
  }, [reload])

  const setStatus = useCallback(async (id: TodoId, status: TodoStatus) => {
    await todoRepository.setStatus(id, status)
    await reload()
  }, [reload])

  const remove = useCallback(async (id: TodoId) => {
    await todoRepository.delete(id)
    await reload()
  }, [reload])

  return { todos, error, create, setStatus, remove }
}
```

No TanStack Query. There is one query and three mutations, all on the
same key — `useState` + reload-after-mutation is enough.

### Form

`TodoForm` keeps a local `expanded` flag:

- **Collapsed:** a single "Add" button (`min-h-11`, accent
  background). See § "Selector Contract" for the required
  `aria-label`.
- **Expanded:** a `<form>` containing
  - `<input type="text">` for the title (autofocus,
    `aria-label="Title"`, `maxLength={200}`, required).
  - `<input type="date">` for the due date
    (`aria-label="Due date"`, required). No `min`: the user is
    allowed to backdate a todo they forgot.
  - A submit button with text "Save" (`type="submit"`).
  - A "Cancel" button (`type="button"`) that collapses the form.

On submit:
1. Read the title and the date string (`YYYY-MM-DD`).
2. Compute `dueAt` as local midnight on that date:
   ```ts
   const [y, m, d] = dateStr.split('-').map(Number)
   const dueAt = new Date(y, m - 1, d).getTime()
   ```
3. Call `create({ title, dueAt })`.
4. On success: clear inputs, collapse, return focus to the Add button.
5. On `ValidationError` (or any thrown `Error`): render the message in
   an inline `<p role="alert">` below the form. The form stays open.

The form is the only place input validation matters in the UI; nothing
else takes user input.

### Row

`TodoRow` renders:

- A native `<input type="checkbox">`, `checked={todo.status === 'completed'}`.
  Toggling it calls `setStatus(todo.id, next)`.
- The title text. Strikethrough + muted colour when `completed`.
- The due date, formatted via `Intl.DateTimeFormat`:
  - Current year: `{ month: 'short', day: 'numeric' }`.
  - Other years: append `{ year: 'numeric' }`.
- A delete button on the right (× glyph or "Delete" label). Calls
  `window.confirm('Delete this todo?')` then `remove(todo.id)`.

The row's primary click target is the checkbox. The delete button is
its own target with `aria-label="Delete <title>"`.

No drag handle, no menu, no edit affordance, no navigation on row click.

### Styling

Tailwind utilities only. `styles.css`:

The palette is **dark grey + white only**. No hues. The system colour
scheme decides which side of the inversion is active.

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --bg:      0 0% 100%;   /* white */
  --surface: 0 0% 97%;    /* very light grey */
  --border:  0 0% 88%;    /* light grey */
  --text:    0 0% 12%;    /* near-black */
  --muted:   0 0% 45%;    /* medium grey */
  --accent:  0 0% 15%;    /* dark grey — primary button / focus ring */
  --danger:  0 0% 40%;    /* mid grey — used for inline errors */
}
@media (prefers-color-scheme: dark) {
  :root {
    --bg:      0 0% 8%;   /* near-black */
    --surface: 0 0% 14%;
    --border:  0 0% 22%;
    --text:    0 0% 98%;  /* white */
    --muted:   0 0% 65%;
    --accent:  0 0% 92%;  /* near-white — primary button */
    --danger:  0 0% 60%;
  }
}

html, body, #root { height: 100%; }

body {
  background: hsl(var(--bg));
  color: hsl(var(--text));
  font-family: ui-sans-serif, system-ui, -apple-system, "Segoe UI",
               Roboto, sans-serif;
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
}
```

The system colour scheme is the source of truth; there is no JS toggle.
`tailwind.config.ts` exposes these tokens as `bg-bg`, `bg-surface`,
`text-text`, `text-muted`, `bg-accent`, etc. (See
§ "Tailwind" for the Tailwind config.)

### Accessibility

- Every form control has a `<label>` or `aria-label`.
- `:focus-visible` ring on every focusable control:
  `focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-accent`.
- The title input is focused automatically when the form expands.
- The checkbox is native and announces its state.
- The delete button has an explicit `aria-label` that includes the
  todo's title.
- The validation error is rendered in `<p role="alert">` so screen
  readers announce it.

### PWA

#### Icons

**PNG icons are mandatory for PWA install.** Chrome's installability
check rejects manifests without PNG icons at 192×192 and 512×512. SVG
manifest entries do not satisfy the check.

The brand mark for v1 is a **solid dark square** (`#1a1a1a` ≈ gray 26).
No bars, no glyph, no SVG anywhere. The mark is decorative; the
deliverable is "QR scans → app installs," and installability requires
PNG, not a particular design. A solid dark tile is the same aesthetic
register as the rest of the app (dark grey + white only) and removes
every transcription risk a richer mark would introduce.

##### How the PNGs land on disk

The icons agent writes one file — `public/icons/make-icons.mjs` (verbatim
from the code block below) — then runs `node public/icons/make-icons.mjs`.
The script emits both PNGs at the canonical paths and exits. No
dependency, no rasterizer, no base64. The PNGs are pure Node output:
PNG signature + IHDR (grayscale, 8-bit) + zlib-deflated IDAT scanlines
+ IEND, with a 256-entry CRC32 table computed inline.

```js
// public/icons/make-icons.mjs
//
// Pure-Node PNG writer for two solid-color icons. Reads nothing,
// pulls no rasterizer dependency. The icons agent writes this file
// from the spec, then runs `node public/icons/make-icons.mjs`. The two
// PNGs land at `public/icons/icon-192.png` and
// `public/icons/icon-512.png`.

import { writeFileSync, mkdirSync } from 'node:fs'
import { deflateSync } from 'node:zlib'
import { dirname } from 'node:path'

const CRC_TABLE = (() => {
  const t = new Uint32Array(256)
  for (let i = 0; i < 256; i++) {
    let c = i
    for (let j = 0; j < 8; j++) c = c & 1 ? 0xedb88320 ^ (c >>> 1) : c >>> 1
    t[i] = c >>> 0
  }
  return t
})()

function crc32(buf) {
  let c = 0xffffffff
  for (let i = 0; i < buf.length; i++) c = CRC_TABLE[(c ^ buf[i]) & 0xff] ^ (c >>> 8)
  return (c ^ 0xffffffff) >>> 0
}

function chunk(type, data) {
  const len = Buffer.alloc(4)
  len.writeUInt32BE(data.length, 0)
  const typeBuf = Buffer.from(type, 'ascii')
  const crc = Buffer.alloc(4)
  crc.writeUInt32BE(crc32(Buffer.concat([typeBuf, data])), 0)
  return Buffer.concat([len, typeBuf, data, crc])
}

function solidPng(size, gray) {
  const sig = Buffer.from([0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a])
  const ihdr = Buffer.alloc(13)
  ihdr.writeUInt32BE(size, 0) // width
  ihdr.writeUInt32BE(size, 4) // height
  ihdr[8] = 8                 // bit depth
  ihdr[9] = 0                 // colour type: grayscale
  ihdr[10] = 0                // compression
  ihdr[11] = 0                // filter
  ihdr[12] = 0                // interlace
  const raw = Buffer.alloc(size * (1 + size))
  for (let y = 0; y < size; y++) {
    raw[y * (1 + size)] = 0   // per-scanline filter byte: None
    for (let x = 0; x < size; x++) raw[y * (1 + size) + 1 + x] = gray
  }
  return Buffer.concat([
    sig,
    chunk('IHDR', ihdr),
    chunk('IDAT', deflateSync(raw)),
    chunk('IEND', Buffer.alloc(0)),
  ])
}

const GRAY = 0x1a // matches `--bg` in dark mode; reads as a dark tile on home screens

for (const size of [192, 512]) {
  const out = `public/icons/icon-${size}.png`
  mkdirSync(dirname(out), { recursive: true })
  writeFileSync(out, solidPng(size, GRAY))
}
console.log('wrote public/icons/icon-192.png and icon-512.png')
```

The two PNGs decode at exact 192×192 / 512×512 dimensions, color type 0
(grayscale), pixel value 26. Chrome's installability check accepts
them.

##### What the icons agent does

1. Write `public/icons/make-icons.mjs` to disk, verbatim from the block
   above.
2. Run `node public/icons/make-icons.mjs` once. The PNGs appear at
   `public/icons/icon-192.png` and `public/icons/icon-512.png`.
3. Exit.

That is the entire icons module. No SVG content of any kind. No
favicon file. The browser tab's icon falls back to its default; this
is acceptable for a personal PWA whose installed surface is the home
screen, not the browser tab.

The PNGs are what `manifest.icons` references (see § "`vite-plugin-pwa`
config" below) and what the apple-touch-icon resolves to (see
§ "`index.html` head"). The pre-flight in `serve:phone` will fetch
each manifest icon URL and require a 200 response — if the PNGs are
missing or empty, the install path fails before the QR prints.

#### `vite-plugin-pwa` config

```ts
VitePWA({
  registerType: 'autoUpdate',
  includeAssets: [
    'icons/icon-192.png',
    'icons/icon-512.png',
  ],
  manifest: {
    name: 'Todos',
    short_name: 'Todos',
    description: 'A simple list of todos with due dates.',
    start_url: '/',
    scope: '/',
    display: 'standalone',
    background_color: '#1a1a1a',
    theme_color: '#1a1a1a',
    icons: [
      // PNG is what Chrome's installability check actually reads.
      { src: '/icons/icon-192.png', sizes: '192x192', type: 'image/png', purpose: 'any' },
      { src: '/icons/icon-512.png', sizes: '512x512', type: 'image/png', purpose: 'any' },
    ],
  },
  workbox: {
    globPatterns: ['**/*.{js,css,html,svg,png,woff2}'],
    navigateFallback: 'index.html',
  },
  devOptions: { enabled: false },
})
```

`purpose: 'any'` is explicit because some Chrome installability checks
require it. No `maskable` variant is shipped; the v1 mark is a solid
square so maskable adds nothing visible. Absence of `maskable` does
not block install on Chrome 96+.

No SVG entries — the manifest is PNG-only. The pre-flight in
`serve:phone` fetches every URL listed under `manifest.icons` and
makes sure each returns 200; keeping the list to PNGs means a passing
pre-flight proves the installability path is good.

No `workbox-window`. No update prompt. No install-prompt button. The
browser surfaces its own install affordance when criteria are met
(Android Chrome) or via Share → Add to Home Screen (iOS Safari).

#### `index.html` head

```html
<meta name="viewport"
      content="width=device-width, initial-scale=1, viewport-fit=cover" />
<meta name="theme-color" content="#1a1a1a" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="mobile-web-app-capable" content="yes" />
<link rel="apple-touch-icon" sizes="192x192" href="/icons/icon-192.png" />
```

The apple-touch-icon points at the 192 PNG produced by the icons
helper (see § "Icons" above). It is the same file the manifest
references — no extra agent work, no extra rasterization step.

### Boundary Rules

Forbidden in `ui/`:

- `import { db } from '@/data/db'` — go through the repository.
- `import Dexie from 'dexie'`.
- Any new abstraction (services, ports, providers) beyond what this
  spec lists.

Allowed:
- `@/data`, `@/domain` (types only), React, Tailwind classes.

### Tests

The smoke test (`scripts/smoke.mjs`, see
§ "Smoke test — `scripts/smoke.mjs`") is the verification oracle for
this layer. Component tests are optional and not required for delivery.

---

## Infrastructure

> Build, dev loop, install flow, smoke test. Everything that isn't
> source code but makes the app run.

### package.json scripts

```jsonc
{
  "scripts": {
    "dev":         "vite",
    "build":       "vite build",
    "preview":     "vite preview --host 0.0.0.0 --port 41730 --strictPort",
    "typecheck":   "tsc --noEmit",
    "serve:phone": "node scripts/serve-phone.mjs",
    "smoke":       "node scripts/smoke.mjs",
    "clean":       "node scripts/free-port.mjs 41730"
  }
}
```

**Ports 41730 (serve) and 42730 (smoke) are project-reserved and
deliberately not Vite's defaults (5173 dev, 4173 preview)** — that is
what lets `free-port.mjs` treat any listener on them as a stale prior
run, with no need to fingerprint the process (§ "Port reclaim —
`free-port.mjs`"). `preview` runs with `--strictPort` so an occupied
port is a hard error rather than a silent bump to the next port.
`clean` (`node scripts/free-port.mjs 41730`) is the manual
port-reclaim escape hatch for the deliverable port; `serve:phone`
frees 41730 automatically on start and `smoke` frees its own 42730, so
you rarely need it by hand.

No `prebuild`. The PNG icons are not generated at build time — they
are emitted by the `icons` agent's helper (`public/icons/make-icons.mjs`,
specified verbatim in § "Icons"). The agent
writes the helper once and runs it once. Nothing rasterizes during
`npm run build`.

### Dependencies

```jsonc
{
  "dependencies": {
    "dexie":     "^4",
    "react":     "^18",
    "react-dom": "^18",
    "ulid":      "^2"
  },
  "devDependencies": {
    "@types/react":          "^18",
    "@types/react-dom":      "^18",
    "@vitejs/plugin-react":  "^4",
    "autoprefixer":          "^10",
    "postcss":               "^8",
    "puppeteer-core":        "^25",
    "qrcode-terminal":       "^0.12",
    "tailwindcss":           "^3",
    "typescript":            "^5",
    "vite":                  "^5",
    "vite-plugin-pwa":       "^0.20"
  }
}
```

Pin majors; let minors and patches float. The total of 4 production
and 11 development dependencies is the entire dependency surface for
this product.

### TypeScript

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### Vite

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { VitePWA } from 'vite-plugin-pwa'
import path from 'node:path'

export default defineConfig({
  plugins: [
    react(),
    VitePWA({ /* see § "`vite-plugin-pwa` config" */ }),
  ],
  resolve: { alias: { '@': path.resolve(__dirname, 'src') } },
  server:  { host: '0.0.0.0', port: 5173 },
  preview: {
    host: '0.0.0.0',
    port: 41730,
    // Vite 5 preview rejects unknown Host headers by default. The
    // cloudflared tunnel rewrites the Host to *.trycloudflare.com,
    // which would trip "Blocked request. This host is not allowed."
    // and the phone sees a blank page after a successful scan.
    // Allowing the suffix keeps the install path green without
    // disabling the check entirely.
    allowedHosts: ['.trycloudflare.com'],
  },
  build:   { target: 'es2022', sourcemap: true },
})
```

### Icons

`scripts/` does not own icon generation. The two PNGs Chrome's
installability check requires (`/icons/icon-192.png`,
`/icons/icon-512.png`) are emitted by `public/icons/make-icons.mjs`, a
pure-Node helper specified in full in § "Icons". The `icons` agent
writes the helper once from the spec
and runs it once. No `prebuild` hook, no puppeteer rasterization, no
external rasterizer dependency. `puppeteer-core` stays in
`devDependencies` because the smoke test uses it; it is not used for
icons.

### Tailwind

```ts
// tailwind.config.ts
import type { Config } from 'tailwindcss'

export default {
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        bg:      'hsl(var(--bg) / <alpha-value>)',
        surface: 'hsl(var(--surface) / <alpha-value>)',
        border:  'hsl(var(--border) / <alpha-value>)',
        text:    'hsl(var(--text) / <alpha-value>)',
        muted:   'hsl(var(--muted) / <alpha-value>)',
        accent:  'hsl(var(--accent) / <alpha-value>)',
        danger:  'hsl(var(--danger) / <alpha-value>)',
      },
    },
  },
  plugins: [],
} satisfies Config
```

`postcss.config.cjs`:

```js
module.exports = { plugins: { tailwindcss: {}, autoprefixer: {} } }
```

`darkMode` is not set; the spec uses `prefers-color-scheme` directly
via CSS variables defined in `src/ui/styles.css`. The Tailwind tokens
above resolve to whichever side of the media query is active.

### Install flow — `serve:phone`

`scripts/serve-phone.mjs` is the **only** install-flow orchestrator
(it imports the shared `freePort` helper from `free-port.mjs`).
It contains the orchestration, the pre-flight validation, and the QR
print. There is no second test script. If the QR prints, the install
path is good; if any pre-flight check fails, the QR never prints and
the implementer sees a one-line error.

**Surfacing the QR — the deliverable is that the *full* QR reaches the
user.** The script renders the QR to stdout via `qrcode-terminal`; how
the main agent gets it in front of the user depends on the runtime
(see § "Deliverable"):

- **At a human terminal:** run it in the **foreground**, attached, with
  stdout not redirected away, until the user terminates with Ctrl-C.
  The QR renders directly to the screen.
- **In an agent / automation harness (no shared terminal):** the main
  agent **captures** the script's stdout, waits for the QR and the
  `*.trycloudflare.com` URL to appear, **reprints the complete QR block
  (every row) and the URL verbatim** in its reply, and leaves the
  process running so the tunnel stays up. Capturing the output is
  required here — it is the only path by which the QR reaches the user.

Either way the QR must appear **in full**. A discarded, piped-away,
truncated, or summarized QR **has not shipped**, and the tunnel must
stay up until the user has installed.

The user-side narrative (which phone browser, install gesture) is in
§ "Phone side — what the user does beyond scanning."
Operator-facing troubleshooting (tunnel caveats, Wi-Fi blocks, etc.)
is in the root `README.md` § "Troubleshooting" and is not spec
material.

#### Port reclaim — `free-port.mjs`

`scripts/free-port.mjs` exports `freePort(port)` and also runs as a
CLI (`node scripts/free-port.mjs 41730`, wired to `npm run clean`).
`freePort` is a **parameterised** helper — the port is an argument,
not a hard-coded constant — so each caller frees only its own port.

**The ports are project-reserved (41730 serve, 42730 smoke), so any
listener is a stale prior run.** Because these are uncommon, fixed
ports — *not* Vite's shared defaults (5173 dev, 4173 preview) — nothing
else on the machine is expected to bind them. `freePort` therefore does
**not** fingerprint the process ("is this one *ours*?"). It frees the
port and lets the caller proceed. This is the whole simplification: the
old `reclaimPort` spawned a PowerShell `Get-CimInstance` per PID and ran
a `vite`/`cloudflared`/project-path heuristic purely to avoid killing an
unrelated app on the *shared* default port 4173. Reserving distinctive
ports removes that need, and with it the per-PID process inspection, the
`{ reclaimed, foreign, free }` contract, and the `foreign`-vs-`free`
branching footgun.

**41730 is freed in exactly one place: `serve-phone.mjs`, as its first
action.** It is the single install-flow orchestrator, run by the main
agent as the **last** step of a run, so all handling of the deliverable
port lives at the end, in the one step that owns serving. `smoke.mjs`
does **not** touch 41730 — it runs against its own dedicated port
**42730** (§ "Smoke test"), calling `freePort(42730)` as its first
action. Decoupling the ports means the smoke gate and the final serve
can never contend for the same port.

`freePort(port)` — returns `Promise<boolean>` (`true` once the port is
bindable, `false` if something still holds it):

1. **Probe first (cheap, common case).** Try to bind the port in-process
   (`net.createServer().listen({ port, host: '0.0.0.0' })`). If it binds,
   close it and return `true` immediately — **no subprocess is spawned
   when the port is already free.** This is the dominant path.
2. **Only if occupied:** find the listener PID(s). Windows: parse
   `netstat -ano -p TCP` (exact-port match on the LISTENING rows); POSIX:
   `lsof -ti tcp:<port> -sTCP:LISTEN`.
3. Kill each PID's whole **tree** — `taskkill /pid <pid> /T /F` on
   Windows; process-group `SIGKILL` on POSIX. Log each killed PID
   (`[free-port] killing pid <pid> on :<port>`) so the action is visible.
4. Poll the bind-probe (≈100 ms × up to ~3 s) until the socket releases.
   Return `true` when bindable, `false` on timeout.

**Canonical primitives — transcribe these two.** The bind-probe and the
`netstat`/`lsof` parse are the bug-prone, non-obvious core (exact-port
match across IPv4/IPv6 forms, LISTENING-only); re-deriving them is what
makes this otherwise-tiny file slow to write. The rest of `freePort` —
wiring probe → `pidsOnPort` → `taskkill /T /F` → re-poll, and the CLI
wrapper — is mechanical and follows the prose above.

```js
import net from 'node:net'
import { execSync } from 'node:child_process'

// Probe: can this port be bound right now? No subprocess. The common path.
function canBind(port) {
  return new Promise((resolve) => {
    const srv = net.createServer()
    srv.once('error', () => resolve(false))
    srv.once('listening', () => srv.close(() => resolve(true)))
    srv.listen({ port, host: '0.0.0.0' })
  })
}

// Listener PIDs on the EXACT port. Match the port as a number (not a
// substring), LISTENING state only, across IPv4 (0.0.0.0:/127.0.0.1:)
// and IPv6 ([::]:) local-address forms.
function pidsOnPort(port) {
  const pids = new Set()
  if (process.platform === 'win32') {
    let out = ''
    try { out = execSync('netstat -ano -p TCP', { encoding: 'utf8' }) } catch { return [] }
    for (const line of out.split(/\r?\n/)) {
      const m = line.trim().match(/^TCP\s+\S+:(\d+)\s+\S+\s+LISTENING\s+(\d+)$/i)
      if (m && Number(m[1]) === port) pids.add(Number(m[2]))
    }
  } else {
    try {
      const out = execSync(`lsof -ti tcp:${port} -sTCP:LISTEN`, { encoding: 'utf8' })
      for (const tok of out.split(/\s+/)) if (tok) pids.add(Number(tok))
    } catch { /* no listener */ }
  }
  return [...pids]
}
```

**Callers guard on the boolean.** The correct usage is:

```js
if (!(await freePort(PORT))) {
  // something still holds the port — abort with the pre-flight message
}
```

There is no array to misread; the single boolean is the whole contract.

This is what makes `--strictPort` safe to keep: a stale preview from a
prior run is freed automatically before the bind, and if `freePort` ever
*can't* clear the port, the subsequent `vite preview --strictPort` fails
fast with `Port <n> is in use` — that hard error **is** the fallback, so
the script never silently bumps to the next port. **Free-on-start, not
teardown-on-exit, is the durable fix** — it does not depend on the
previous run having shut down cleanly (a hard kill, a closed terminal,
or a closed editor / agent session all skip exit teardown; the next run
frees the port regardless).

#### Behaviour

0. **Free the port.** Before anything else, `if (!(await
   freePort(41730))) abort` (§ "Port reclaim"). After this, 41730 is
   bindable; if `freePort` returned `false` something still holds it, so
   abort immediately with `[serve:phone] pre-flight failed: port 41730 is
   occupied by another process — stop it and retry` and never build or
   open the tunnel.
1. **Build.** Always run `npm run build` as a subprocess and wait
   for exit 0 before continuing. Stream its output. If build fails,
   exit non-zero with the build's exit code and a one-line hint. The
   user may have edited the spec since the last run — never serve a
   stale `dist/`.
2. **Boot preview.** `spawn('npm', ['run', 'preview'])` (the project's
   own preview script, so port/host stay consistent). The preview
   script runs `vite preview --port 41730 --strictPort` so an occupied
   port is a hard error instead of a silent bump to the next port. Step
   0 already freed the port, so if the preview child still emits `Port
   41730 is in use` (or never reports listening on 41730 within the
   timeout), abort with `[serve:phone] pre-flight failed: port 41730 is
   occupied by another process — stop it and retry` and never open the
   tunnel. The tunnel and pre-flight target the same port the preview
   actually bound.
3. **Wait for localhost.** Poll `http://localhost:41730/` with `fetch`
   until it returns 200 — **the fetch is the sole readiness signal.**
   Time out at 30 s with a clear error. Do **not** gate readiness on
   parsing the preview's stdout: Vite prints its `Local:` line with ANSI
   color codes (`localhost:\x1b[1m41730\x1b[22m`), so a literal
   `localhost:41730` substring match is unreliable and has falsely failed
   otherwise-healthy runs with a 30 s timeout. You may still watch stdout
   for a `Port 41730 is in use` line as an early fail-fast per step 2, but
   the *absence* of a ready line is never itself a failure — only the
   `fetch` timeout is.
4. **Pre-flight.** Run, against `http://localhost:41730`, in order:
   - **`GET /`** → 200, body contains `<link rel="manifest"`.
   - **`GET /manifest.webmanifest`** → 200, response is JSON, parsed
     manifest contains:
     - non-empty `name`,
     - `start_url`,
     - `display: 'standalone'`,
     - `icons` array with at least one entry whose `sizes` includes
       `192x192` and at least one whose `sizes` includes `512x512`.
   - **For each icon in `manifest.icons`:** `GET <resolved-url>` → 200.
   - Any failure: print `[serve:phone] pre-flight failed: <reason>`,
     send `SIGTERM` to the preview child, exit code 1. Do not start
     the tunnel.
5. **Boot tunnel.** `spawn('cloudflared', ['tunnel', '--url',
   'http://localhost:41730'])`.
6. **Capture URL.** Pipe cloudflared's stdout and stderr. On each
   line, test against the regex
   `https:\/\/[a-z0-9.-]+\.trycloudflare\.com`. On the first match,
   capture the URL and set a `printed` flag. Subsequent matches in
   the same run are dropped (the user sees one QR, not several).
7. **Print, exactly once per run:**
   - A blank line.
   - The QR rendered by `qrcode-terminal`:
     `qrcode.generate(url, { small: true })`.
   - The URL on its own line, in case the QR cannot be scanned.
   - A two-line phone reminder:
     ```
     iOS:     Safari → Share → Add to Home Screen
     Android: Chrome → Install banner, or menu → Install app
     ```
8. Forward all child output to this process's stdout/stderr.
9. **Teardown — kill the whole tree.** Spawn the preview and tunnel so
   the whole tree can be torn down: track each child's PID. On `SIGINT`
   / `SIGTERM` (and on any fatal path that calls the script's
   `fail()`), kill the **process tree**, not just the direct child:
   - **Windows:** `spawn('taskkill', ['/pid', String(child.pid), '/T',
     '/F'])` — `/T` kills the child and all descendants.
   - **POSIX:** spawn the child with `detached: true`, then
     `process.kill(-child.pid, 'SIGTERM')` to signal the whole process
     group.

   The intended stop is **Ctrl-C in the foreground terminal**; after
   it, no `vite`/`cloudflared`/`node` process from this run remains and
   port 41730 is free.

**Canonical pre-flight (step 4) — transcribe this.** The icon-size
matching (`sizes` is a space-separated token list, not a substring) and
the relative-URL resolution against the manifest are the parts that are
easy to get subtly wrong; the surrounding orchestration (build, spawn,
capture, QR) is standard and stays prose. Throw a one-line reason; the
caller prints `[serve:phone] pre-flight failed: <reason>`, SIGTERMs the
preview, and exits 1 before any tunnel opens.

```js
async function preflight(base) {
  const root = await fetch(base + '/')
  if (!root.ok) throw new Error(`GET / -> ${root.status}`)
  if (!/<link[^>]+rel=["']?manifest/i.test(await root.text()))
    throw new Error('index.html has no <link rel="manifest">')

  const res = await fetch(base + '/manifest.webmanifest')
  if (!res.ok) throw new Error(`manifest -> ${res.status}`)
  let m
  try { m = JSON.parse(await res.text()) } catch { throw new Error('manifest is not valid JSON') }
  if (!m.name) throw new Error('manifest.name is empty')
  if (!m.start_url) throw new Error('manifest.start_url missing')
  if (m.display !== 'standalone') throw new Error("manifest.display must be 'standalone'")

  const icons = m.icons ?? []
  const has = (s) => icons.some((i) => String(i.sizes || '').split(/\s+/).includes(s))
  if (!has('192x192')) throw new Error('manifest has no 192x192 icon')
  if (!has('512x512')) throw new Error('manifest has no 512x512 icon')

  for (const icon of icons) {
    const url = new URL(icon.src, base + '/manifest.webmanifest').href
    const r = await fetch(url)
    if (!r.ok) throw new Error(`icon ${icon.src} -> ${r.status}`)
  }
}
```

   **Exit teardown is best-effort; free-on-start is the guarantee.**
   A hard kill of this process — or closing the terminal, editor, or
   agent session that owns it — skips these handlers, and on Windows
   the `vite`/`cloudflared` grandchildren are not in a kill-on-close
   job, so they can survive as orphans holding 41730. That is tolerated:
   step 0's `freePort` clears them on the next run before binding.
   Do **not** rely on exit teardown alone to keep successive runs
   unblocked — that is exactly the assumption that left the port busy
   before.

The script must work on Windows (PowerShell), macOS, and Linux. Use
`shell: true` when spawning `npm run preview` and `npm run build` so
Windows resolves the `.cmd` shim correctly; use `shell: false` for
`cloudflared` so signals propagate cleanly.

Required external: the `cloudflared` binary must be on the user's PATH
(`winget install Cloudflare.cloudflared` on Windows; `brew install
cloudflared` on macOS; release binary on Linux). The spec does not
bundle cloudflared. If `cloudflared` is missing, the spawn fails and
the script exits with the spawn's error and a one-line install hint.

#### Why pre-flight, not a separate `verify:install` script

Earlier drafts of this spec specified a second script,
`verify-install.mjs`, that ran against the public tunnel URL. It was
removed because:

- The implementer would write two install-flow scripts in a one-shot
  run instead of one.
- The checks it ran (manifest reachable, icons 200, root references
  manifest) all pass identically against `localhost:41730` — there is
  no `trycloudflare.com`-specific failure mode that matters at this
  scope (service-worker registration works on `localhost` and on
  HTTPS tunnel equally).
- Running them as a pre-flight in `serve:phone` means a failure
  surfaces *before* the QR prints, which is exactly when the
  implementer wants to know.

If a future failure mode appears that only manifests over the public
tunnel (e.g. a CSP that breaks on real hostnames), a second oracle
can be reintroduced. Until then, one script is enough.

### Smoke test — `scripts/smoke.mjs`

The smoke test is the verification oracle for success criteria #1–#9
in § "Success Criteria". It **owns its own preview lifecycle on a
dedicated port — 42730, never 41730**: on start it `if (!(await
freePort(42730))) abort`, spawns its own preview bound to 42730 (`vite
preview --host 0.0.0.0 --port 42730 --strictPort` directly — *not* `npm
run preview`, which is hard-wired to the deliverable port 41730), waits
for 42730 to answer 200, runs the puppeteer steps below, and on exit
(success, failure, or signal) tears the preview down with the same
tree-kill as `serve-phone.mjs`. It never frees or binds 41730 — the
deliverable port belongs to the final serve step alone (§ "Port
reclaim"). The orchestrator must **not** start a preview for it — there
is no `npm run preview &` step, and that stray ampersand was the orphan
that left a port busy.

**Selector contract.** Every selector and assertion below maps 1:1 to
the table in § "Selector Contract." That
table is the single source of truth for the strings, aria-labels,
and DOM shapes this script depends on. If the script and the
contract disagree, the contract wins and the script is wrong.

Behaviour:

0. `if (!(await freePort(42730))) abort`, then spawn `vite preview
   --host 0.0.0.0 --port 42730 --strictPort`, poll
   `http://localhost:42730/` until 200 (30 s timeout), and register a
   tree-kill teardown that fires on every exit path. (Port 42730, never
   41730 — see § "Port reclaim".)
1. Launch `puppeteer-core` against the system Chrome.
   Default path on Windows:
   `C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe`.
   Override with `CHROME_PATH`.
2. `page.goto('http://localhost:42730')`.
3. Delete IndexedDB `todos-app` and reload, so each run is
   deterministic.
4. Wait for the Add button (see Selector Contract).
5. Click the Add button. Type `smoke-test-todo` into the Title input.
   Set the Due-date input value to today + 7 days and fire `change`.
   Click Save.
6. Assert a row with title `smoke-test-todo` appears in the list (the
   title `<span>` per the contract).
7. Click that row's checkbox (per the contract: `aria-label` starts
   with `Mark "smoke-test-todo"`). Assert the title span now has the
   `line-through` class.
8. Click the row's delete button (`aria-label='Delete
   "smoke-test-todo"'`). Auto-accept the `window.confirm` dialog.
   Assert the row is gone.
9. Reload. Assert the empty-state text `Nothing to do.` is visible
   again (persistence held: the delete survived reload).
10. Exit 0 if every step held; exit 1 otherwise. Any `pageerror` or
    non-`ERR_ABORTED` `requestfailed` fails the run.

**Canonical step 5 — the React-controlled date input. Transcribe this.**
A raw `el.value = …` is overwritten on React's next render, so the typed
date never reaches state and the todo is created with the wrong (or no)
due date — a silent smoke failure. Set via the native prototype setter,
then dispatch the events React listens for. This is the one puppeteer
step with a non-obvious form; every other step follows directly from the
Selector Contract and needs no canonical block.

```js
await page.$eval(
  'input[aria-label="Due date"]',
  (el, value) => {
    const set = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set
    set.call(el, value)
    el.dispatchEvent(new Event('input', { bubbles: true }))
    el.dispatchEvent(new Event('change', { bubbles: true }))
  },
  dueDate, // 'YYYY-MM-DD', today + 7 days
)
```

The script is the contract between the spec and any implementer:
"does the prototype work?" is answered by this exit code. The
selectors above are exact and must match § "Selector Contract"
verbatim.

### Quality gates

| Gate | Command | Required |
|---|---|---|
| Types | `npm run typecheck` | Must pass |
| Build | `npm run build` | Must pass |
| Smoke | `npm run smoke` (boots and tears down its own preview) | Must pass |
| Install flow | `npm run serve:phone` prints a QR | Must pass |
| Bundle size | check `dist/` | < 250 KB gzipped JS |
| Lighthouse PWA | manual, Chrome DevTools | "Installable" — PNG icons present, SW registered, manifest valid. If Lighthouse marks the site "Not installable" the manifest icons are almost certainly still SVG; check `dist/icons/*.png`. |

Vitest, Playwright, ESLint, and Prettier are **not** required to ship
v1. If the implementer wants them, they may add them; they are not
delivery blockers.

### CI (suggested, not required)

```yaml
# .github/workflows/ci.yml (sketch)
name: ci
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run typecheck
      - run: npm run build
      - run: npm run smoke   # boots and tears down its own preview
```

### Boundary Rules

This module owns build, dev-loop, and verification. It does not own
runtime behaviour. If `serve-phone.mjs` or `smoke.mjs` ever needs to
import application code, it is doing too much.
