# Archived HTML sketch

Superseded by consolidation.md. Text and code extracted from the discarded HTML; interactive controls are omitted. These were illustrative APIs, not shipped code.

# One app.
Plugins all the way up.

surface-cordis-app composes the runtimes.
Your bundle chooses the shell, transports and domain plugins.

Proposed usage, not runnable code. The new package, helpers and lifecycle promises below are API sketches. The defineSurface example uses the existing API. App-specific store and view implementations are omitted.

## Start the application

```tsx
import { defineApp } from "@kolu/surface-cordis-app"
import { web, mcp } from "@kolu/surface-cordis-app/plugins"
import { workspaceShell, inspector } from "@example/workspace-shell"
import { notes } from "./plugins/notes"

const domain = [notes]

export const app = defineApp({
  profiles: {
    web:      [...domain, workspaceShell, inspector, web({ port: 3000 }), mcp()],
    headless: [...domain, mcp()],
  },
})

await app.start({ profile: "web" })
```

The app host composes plugins. The workspace shell is a plugin selected by the bundle.

## Keep the ordinary Surface contract

```tsx
import { Schema } from "effect"
import { defineSurface } from "@kolu/surface/define"

export const notesSurface = defineSurface({
  cells: {
    title: { schema: Schema.String, default: "Untitled" },
  },
})
```

This defineSurface shape already exists. Real notes would add collections and procedures here.

## Declare the plugin’s separate entry points

```tsx
import { definePlugin } from "@kolu/surface-cordis-app"
import { notesSurface } from "./wire"

export const notes = definePlugin({
  id: "notes",
  surface: notesSurface,

  server:  () => import("./server"),
  browser: () => import("./browser"),
})
```

Proposed build contract: the host resolves each entry into its own server/browser module graph. Headless never loads the browser entry.

## Supply the server implementation

```tsx
import { defineServer } from "@kolu/surface-cordis-app/server"
import { makeNotesDeps } from "./store"

export default defineServer({
  // Effect<ImplementSurfaceDeps<typeof notesSurface>, Error, Scope>
  setup: makeNotesDeps,

  expose: {
    browser: { title: "resource" },
    mcp:     { title: "resource" },
  },
})

// Host: initialize → mount Surface → publish availability.
// Unload: withdraw → settle owned work → release the scope.
```

makeNotesDeps is the application’s Effect that acquires its store and returns Surface dependencies. The host owns the mount, exposure and teardown coordination.

## Depend on a shell only where you use it

```tsx
import { defineBrowser } from "@kolu/surface-cordis-app/browser"
import { Workspace } from "@example/workspace-shell/contracts"
import { NotesView } from "./NotesView"

export default defineBrowser({
  needs: { workspace: Workspace },

  mount({ client, workspace }) {
    // client is the plugin’s ready Surface client.
    return workspace.panel({
      id: "notes",
      title: "Notes",
      render: () => <NotesView client={client} />,
    })
  },
})

// panel() returns an owned disposer.
// No Workspace provider → this browser component waits.
// Notes’ server component has no Workspace dependency.
```

Workspace and panel() belong to the shell plugin. The app framework only understands dependencies and owned contributions.

```tsx
// Inside NotesView: ordinary Surface hooks.
const title = client.cells.title.use({
  authority: "server",
  onError: error => console.error("notes.title", error),
})

return <h1>{title.value()}</h1>
```

## The shell is an ordinary provider

```tsx
import { defineBrowser } from "@kolu/surface-cordis-app/browser"
import { Workspace } from "./contracts"
import { mountWorkspace } from "./ui"

export default defineBrowser({
  mount({ root, provide, own }) {
    const workspace = mountWorkspace(root)
    own(workspace.dispose)

    provide(Workspace, {
      panel: workspace.registerPanel,
    })
  },
})

// The bridge withdraws Workspace and joins dependent cleanup
// before disposing the workspace. Cordis owns dependency resolution.
```

The shell implements its layout and registration service. Surface has no panel or slot API. Settings and inspector UIs are consumers of shell services too.

## Changing the app is one operation

```tsx
await app.plugins.disable("notes")
// Its browser contribution, wire availability and MCP catalog retract.
// Unrelated plugins and their local UI state keep running.

await app.plugins.enable("notes")
// Fresh server activation; clients follow the new mount.
// Browser contribution activates after its dependencies are ready.

await app.plugins.disable("workspace-shell")
// Dependent UI contributions unload. Notes’ server stays running.

await app.stop()
// Join plugin cleanup, then close the host.
```

These methods express a proposed settled-lifecycle contract. Failure is surfaced; a failed or hanging finalizer is never reported as successful cleanup.

## What we actually have to build.

First proof: extract one real Olai capability into this shape. Run it with a shell, then headless. Disable and re-enable it without rebuilding unrelated UI. Let that slice determine the API.

Root-level contributions and stronger independent-plugin contract checks remain candidates; neither is assumed necessary for this first slice. The permanent host must stay free of shell-specific contracts.

Already merged: live mounts #2223 · live accepts #2225 · stable clients #2228 · live header policy #2229.

Extraction seed: Olai’s Effect–Cordis bridge. Composition gap: Olai’s browser coordination. Theory: Cordis paper. Disposing registrations does not undo emitted writes or messages.

Notes stay in this open tab. Export before closing or reloading.

Feedback export requires JavaScript. All examples remain readable.
