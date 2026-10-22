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
  belongs_to :bucket, touch: true

  broadcasts_refreshes
end
```

```erb
<%# Any page that should stay live subscribes: %>
<%= turbo_stream_from @message %>

<%# Or subscribe to the parent to refresh on any child change: %>
<%= turbo_stream_from @bucket %>
```

The browser re-fetches the page and morphs only the changed DOM nodes — scroll, focus, and unchanged elements are preserved. This replaces most hand-written `*.turbo_stream.erb` files.

- Exclude client-managed elements from morphing with `data-turbo-permanent`.
- Reach for explicit streams only when a full-page refresh is genuinely too coarse.

## Turbo Frames for Modals and Lazy Loading

```erb
<%# Index page with an empty target frame %>
<%= turbo_frame_tag :modal %>

<%# Link that loads into the frame %>
<%= link_to "Edit", edit_project_path(@project),
    data: { turbo_frame: :modal } %>

<%# edit.html.erb — content targeted at the frame %>
<%= turbo_frame_tag :modal do %>
  <dialog data-controller="dialog">
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
- Vanilla CSS (nesting and custom properties cover most of what preprocessors did).

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
