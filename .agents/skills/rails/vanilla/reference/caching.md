# Caching

In 37signals apps caching shapes the design: decide early what can be cached and for whom, because that decides where per-user logic is allowed to live. These are standard Rails tools; this is how they fit together.

## Touch Chains

Cache keys include `updated_at`, so changing a record invalidates every fragment keyed on it. `touch: true` pushes the change up to the parents:

```ruby
module Recordable
  extend ActiveSupport::Concern

  included do
    has_one :recording, as: :recordable, touch: true
  end
end

class Recording < ApplicationRecord
  belongs_to :bucket, touch: true
end
```

Editing a message now expires the fragments for the message, its recording, and its bucket, with no manual sweeping. State records touch their parent the same way (`Publication` in `domain-modeling.md`).

## Fragment Caching

```erb
<%# Nested ("Russian doll"): the outer fragment is reused until something inside changes %>
<% cache @bucket do %>
  <%= render partial: "recordings/recording", collection: @bucket.recordings, cached: true %>
<% end %>

<%# Put everything that changes the output in the key %>
<% cache [ recording, Current.person.admin? ] do %>
  <%= render "recordings/admin_tools", recording: recording %>
<% end %>
```

`cached: true` fetches all of a collection's fragments in one read instead of one per item.

The digest in a fragment's key covers the template and the partials it renders, but not helpers or partials rendered inside helpers. Declare such partials with a `Template Dependency:` comment, and when a helper changes, bump a `Helper Dependency Updated:` comment (`comments.md`).

## Keep Per-User Content Out of Shared Fragments

A fragment cached once for everyone can't contain this person's "Edit" link, a "(you)" label, or their local time. Two ways around it:

**1. Render for everyone, adjust in the browser.** Put the current person in a meta tag and let a small Stimulus controller reveal what depends on it:

```erb
<%# layout, in <head> %>
<meta name="current-person-id" content="<%= Current.person&.id %>">

<%# inside a shared cached fragment %>
<div data-controller="ownership" data-ownership-creator-id-value="<%= recording.creator_id %>">
  <%= link_to "Edit", edit_recording_path(recording), hidden: true, data: { ownership_target: "ownerOnly" } %>
</div>
```

```javascript
// app/javascript/controllers/ownership_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = [ "ownerOnly" ]
  static values = { creatorId: String }

  connect() {
    const currentPersonId = document.head.querySelector("meta[name=current-person-id]")?.content

    if (currentPersonId === this.creatorIdValue) {
      this.ownerOnlyTargets.forEach(element => element.hidden = false)
    }
  }
}
```

This only changes what's displayed. The server still authorizes the action (`domain-modeling.md`).

**2. Load the personal part separately.** Leave a lazy Turbo Frame in the cached page and serve the per-user piece from its own small request:

```erb
<%= turbo_frame_tag :my_menu, src: my_menu_path, loading: :lazy, refresh: :morph %>
```

## HTTP Caching

```ruby
class RecordingsController < ApplicationController
  def show
    @recording = Current.account.recordings.find(params[:id])
    fresh_when etag: [ @recording, Current.person ]
  end
end
```

`fresh_when` answers `304 Not Modified` when the ETag matches, so nothing renders at all. Include everything that changes the response in the ETag: the record, the viewer, anything else from `Current` that the page shows. For something every action depends on, declare it once with `etag { ... }` at the controller level.

Skip `fresh_when` on pages that contain forms: a `304` reuses the browser's copy, and its embedded CSRF token can be stale after the session changes.
