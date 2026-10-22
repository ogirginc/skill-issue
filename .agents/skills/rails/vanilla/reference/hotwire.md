# Hotwire (Turbo + Stimulus)

Progressive enhancement with server-rendered HTML. Decision order for dynamic UI:

1. **Full page + Turbo Drive** — often enough; don't reach further without need.
2. **Morphing page refreshes (Turbo 8)** — the default for "keep this page live". Broadcast a refresh, let morph diff the DOM.
3. **Turbo Frames** — scoped navigation, modals, lazy loading.
4. **Turbo Streams** — surgical multi-element updates when morphing doesn't fit (append to a list without re-rendering the page, cross-page broadcasts of a single element).
5. **Stimulus** — sprinkles of client behavior on top.

## Morphing Refreshes (Turbo 8) — Start Here

```erb
<%# app/views/layouts/application.html.erb, in <head> %>
<%= turbo_refreshes_with method: :morph, scroll: :preserve %>
```

```ruby
# Model broadcasts a refresh to subscribers whenever it changes
class Message < ApplicationRecord
  belongs_to :bucket, touch: true # keeps the bucket's cache keys fresh (see caching.md)

  # Pages subscribed to this message (updates, destroys)
  broadcasts_refreshes
  # Pages subscribed to its bucket (any change, including new messages)
  broadcasts_refreshes_to :bucket
end
```

```erb
<%# Any page that should stay live subscribes: %>
<%= turbo_stream_from @message %>

<%# Or subscribe to the parent to refresh on any child change: %>
<%= turbo_stream_from @bucket %>
```

`broadcasts_refreshes` alone never reaches a page subscribed to the parent: it sends creates to a stream named after the model (`"messages"`, shared by every tenant — see Morphing Pitfalls) and updates/destroys to the record itself. Pages that show the parent need `broadcasts_refreshes_to :parent`.

The browser re-fetches the page and morphs only the changed DOM nodes — scroll, focus, and unchanged elements are preserved. This replaces most hand-written `*.turbo_stream.erb` files.

- Exclude client-managed elements from morphing with `data-turbo-permanent`.
- Reach for explicit streams only when a full-page refresh is genuinely too coarse.

### Morphing Pitfalls

- **Unique DOM IDs.** Morphing matches elements by `id`, so duplicates (the same partial rendered twice with `dom_id(record)`) update the wrong node. Add a prefix: `dom_id(record, :preview)`.
- **Lazy frames.** On a morph refresh, a frame with `src` loses the content it loaded unless it's flagged `refresh: :morph`; with the flag, Turbo reloads the frame and morphs it instead:
  ```erb
  <%= turbo_frame_tag :notifications, src: notifications_path, loading: :lazy, refresh: :morph %>
  ```
- **Client-side state.** A morph resets whatever JS put into the DOM (formatted local times, restored drafts). Re-apply it on the element's `turbo:morph-element` event, which fires on each morphed element: `data-action="turbo:morph-element->local-time#refresh"`.
- **Open dialogs.** A morph also removes attributes the server didn't render, including `open` on a dialog the user opened, which closes it. Cancel that change on `turbo:before-morph-attribute`, as the dialog controller below does (Fizzy's does the same).
- **Streams can morph too.** `turbo_stream.replace(target, partial: "...", method: :morph)` updates in place instead of swapping the element, which avoids flicker and keeps focus and unchanged nodes.
- **Scope collection-wide streams by account.** A stream named without a record (`turbo_stream_from :messages`) is shared by every tenant, so a change in one account refreshes every subscriber in all of them. Fizzy fixed this in PR #1800 ("Scope general broadcasts by account"):
  ```ruby
  broadcasts_refreshes_to ->(message) { [ message.account, :messages ] }
  ```
  ```erb
  <%= turbo_stream_from Current.account, :messages %>
  ```
  Streams for a record (`turbo_stream_from @bucket`) are already specific to it.

## Turbo Frames for Modals and Lazy Loading

```erb
<%# Index page with an empty target frame %>
<%= turbo_frame_tag :modal %>

<%# Link that loads into the frame %>
<%= link_to "Edit", edit_project_path(@project),
    data: { turbo_frame: :modal } %>

<%# edit.html.erb — content targeted at the frame %>
<%= turbo_frame_tag :modal do %>
  <dialog data-controller="dialog"
          data-action="turbo:before-morph-attribute->dialog#preventCloseOnMorphing">
    <%= render "form", project: @project %>
  </dialog>
<% end %>
```

```javascript
// app/javascript/controllers/dialog_controller.js
// <dialog open> renders non-modal — call showModal() for backdrop + Esc-to-close.
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    this.element.showModal()
  }

  close() {
    this.element.close()
  }

  // A morph refresh would remove `open`, which the server never renders
  preventCloseOnMorphing(event) {
    if (event.detail?.attributeName === "open") {
      event.preventDefault()
    }
  }
}
```

## Turbo Streams for Surgical Updates

```ruby
# Controller
def create
  @message = @bucket.messages.create!(message_params)

  respond_to do |format|
    format.turbo_stream
    format.html { redirect_to @bucket }
  end
end
```

```erb
<%# create.turbo_stream.erb %>
<%= turbo_stream.append :messages do %>
  <%= render @message %>
<% end %>

<%= turbo_stream.update :message_count, @bucket.messages.count %>

<%= turbo_stream.remove :new_message_form %>
```

## Stimulus Controllers

```javascript
// app/javascript/controllers/position_controller.js
import { Controller } from "@hotwired/stimulus"
import { post } from "@rails/request.js"

export default class extends Controller {
  static values = { url: String }

  async reposition(event) {
    const position = this.#calculatePosition(event)
    await post(this.urlValue, {
      body: JSON.stringify({ position })
    })
  }

  #calculatePosition(event) {
    // Position calculation logic
  }
}
```

```erb
<div data-controller="position"
     data-position-url-value="<%= position_path(@item) %>">
  <%= yield %>
</div>
```

Keep Stimulus controllers small, generic, and reusable (a `dialog` controller, not a `project_edit_modal` controller). State lives in the DOM via values/classes/targets, not in JS objects.

## No-Build Frontend

Default to the no-build setup — it removes an entire class of tooling problems:

- **importmap-rails** for JS: pin dependencies, ship ES modules as-is, no bundler, no `node_modules`.
- **Propshaft** for assets: digest-and-serve, no compilation step.
- Native CSS: nesting and custom properties cover most of what preprocessors did. Fizzy's Gemfile has no Sass, Tailwind, or JS bundler — only importmap-rails and Propshaft.

```ruby
# config/importmap.rb
pin "@rails/request.js", to: "requestjs.js"
pin_all_from "app/javascript/controllers", under: "controllers"
```

Only add a bundler when a dependency truly requires a build step — and prefer dependencies that don't.

## Key Principles

- **Progressive enhancement** - Works without JS, enhanced with it
- **Server renders HTML** - No client-side templates, no JSON-driven UI
- **Morph first** - Broadcast refreshes; hand-write streams only when needed
- **Stimulus** - Sprinkles of JS behavior, not SPA architecture
