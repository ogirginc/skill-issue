---
name: vanilla
description: Apply 37signals (Basecamp/HEY/Fizzy) vanilla Rails patterns and code style including delegated types, RESTful controllers instead of custom actions, model-predicate authorization, Hotwire (Turbo morphing/frames/streams, Stimulus), CurrentAttributes with callbacks, state-as-records, caching, fixtures-based testing, and no-build frontend. Use when building Rails features, designing models, implementing polymorphic content, creating real-time UI, or structuring domain logic. Triggers on "vanilla Rails", "Basecamp-style", "37signals patterns".
---

# 37signals Rails Patterns

This skill applies 37signals architectural patterns from Basecamp, HEY, and Fizzy to Rails development. These patterns emphasize pragmatism over dogma, rich domain models, and leveraging Rails conventions.

**When to defer:** In an existing codebase, its conventions win. If the app already uses service objects, RSpec with factories, or a JS framework, match that style — apply these patterns to new subsystems or greenfield work, not as drive-by rewrites of working code.

**Code style:** before writing Ruby, read `reference/style.md` — 37signals' own conventions (expanded conditionals, method order, `private` indentation, `!`, `_later`/`_now`).

## Core Principles

1. **Vanilla Rails is plenty** - No separate service/application layers. Rich domain models with strategic delegation.
2. **Sharp knives philosophy** - Powerful tools (callbacks, globals) used with discipline.
3. **Concerns for organization** - Group related functionality, not arbitrary layers.
4. **REST all the way down** - New controller instead of custom action.
5. **Progressive enhancement** - Server renders HTML; Turbo and Stimulus enhance it.

## Pattern Index

Read the matching reference file before applying a pattern — each contains the full code and pitfalls.

| Need | Pattern | Reference |
|------|---------|-----------|
| Multiple content types, shared operations | Delegated types + capabilities | `reference/delegated-types.md` |
| Parent-child relationships | Tree concern | `reference/tree-positioned.md` |
| Ordering/positioning | Positioned concern | `reference/tree-positioned.md` |
| Sign-in, sessions | Rails 8 authentication generator + `Current.session` | `reference/current-attributes.md` |
| Request context in models | CurrentAttributes | `reference/current-attributes.md` |
| Auto-create associated records, audit trail | Callbacks + Eventable | `reference/current-attributes.md` |
| Rich models, complex operations, concerns | Domain modeling | `reference/domain-modeling.md` |
| N+1 queries on list pages | `preloaded` scope | `reference/domain-modeling.md` |
| Verb-like feature ("publish", "copy", "archive") | RESTful sub-resource controller | `reference/domain-modeling.md` |
| Sub-resource controllers sharing a parent lookup | Scoping controller concern | `reference/domain-modeling.md` |
| Permission or capability checks | Model predicates + `before_action` → `head :forbidden` | `reference/domain-modeling.md` |
| Boolean flag that needs who/when | State as records | `reference/domain-modeling.md` |
| Background work | Thin jobs calling model methods (`_later` / `_now`) | `reference/domain-modeling.md` |
| Jobs that need the current tenant | `AccountTenanted` job extension | `reference/current-attributes.md` |
| Fragment/HTTP caching, per-user content in cached pages | Touch chains, `cache`, `fresh_when` | `reference/caching.md` |
| Real-time UI updates | Turbo morphing refreshes, then streams | `reference/hotwire.md` |
| Scoped page sections, modals, lazy loading | Turbo Frames | `reference/hotwire.md` |
| JS behavior sprinkles | Stimulus controllers | `reference/hotwire.md` |
| Asset pipeline / JS setup | No-build (importmaps + Propshaft) | `reference/hotwire.md` |
| Tests for any of the above | Minitest + fixtures | `reference/testing.md` |

## File Organization

```
app/models/
├── recording.rb                 # Main model
├── recording/
│   ├── copyable.rb             # Concern
│   ├── copier.rb               # Complex operation
│   ├── incineratable.rb        # Concern
│   └── incineration.rb         # Complex operation
├── concerns/
│   ├── tree.rb                 # Reusable across models
│   ├── positioned.rb
│   └── eventable.rb
└── current.rb                   # CurrentAttributes

app/controllers/
├── recordings_controller.rb     # CRUD only
├── concerns/
│   └── recording_scoped.rb      # Shared parent lookup for sub-resources
└── recordings/
    ├── copies_controller.rb     # POST /recordings/:id/copies
    └── incinerations_controller.rb

app/jobs/
├── concerns/
│   └── account_tenanted.rb      # Restores Current.account in jobs
└── recording/
    └── incineration_job.rb      # Calls recording.incinerate_now
```

## Anti-Patterns to Avoid

1. **Service objects for everything** - Put behavior on domain models
2. **Anemic models** - Models should have behavior, not just data
3. **Separate application layer** - Controllers call models directly
4. **Custom controller actions** - Add a sub-resource controller instead
5. **Client-side rendering** - Server renders HTML, Turbo delivers it
6. **Hand-written Turbo Streams by default** - Start with morphing refreshes; streams are for surgical cases
7. **Rewriting working code** - Enhance progressively instead
8. **Avoiding callbacks dogmatically** - Use them for orthogonal concerns
9. **Passing request context everywhere** - Use CurrentAttributes
10. **Boolean state columns** - When you need who/when, use a record (see state-as-records)
11. **Unscoped lookups** - Find through the account or what the person can access, never `Model.find(params[:id])`
12. **Policy objects for simple rules** - Permission predicates on models, checked in a `before_action`
