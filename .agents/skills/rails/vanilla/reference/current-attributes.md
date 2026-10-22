# CurrentAttributes + Callbacks

Thread-safe request context combined with callbacks for orthogonal concerns.

## Current Class

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :account, :person, :request_id
  attribute :user_agent, :ip_address

  def person=(person)
    super
    self.account = person&.account
  end
end
```

## Setting Current (in ApplicationController)

```ruby
class ApplicationController < ActionController::Base
  before_action :set_current_attributes

  private

  def set_current_attributes
    Current.person = Person.find_by(id: session[:person_id])
    Current.request_id = request.request_id
    Current.user_agent = request.user_agent
    Current.ip_address = request.remote_ip
  end
end
```

## Using Current in Models

```ruby
class Project < ApplicationRecord
  belongs_to :account, default: -> { Current.account }
  belongs_to :creator, class_name: "Person", default: -> { Current.person }
end

class Recording < ApplicationRecord
  belongs_to :creator, default: -> { Current.person }
end
```

## Callbacks for Orthogonal Concerns

```ruby
# Auto-create associated records
module Bucketable
  extend ActiveSupport::Concern

  included do
    has_one :bucket, as: :bucketable, dependent: :destroy
    after_create :create_default_bucket, unless: :bucket
  end

  private

  # Must not be named create_bucket! — that would override the has_one-generated
  # method and recurse into itself.
  def create_default_bucket
    create_bucket!(account: account)
  end
end

# Event tracking with request context
module Eventable
  extend ActiveSupport::Concern

  included do
    has_many :events, dependent: :destroy
    after_create :track_created
  end

  def track_event(action, particulars = {})
    events.create!(
      action: action,
      creator: Current.person,
      request_id: Current.request_id,
      user_agent: Current.user_agent,
      ip_address: Current.ip_address,
      **particulars
    )
  end

  private

  def track_created
    track_event(:created)
  end
end
```

## Suppress for Exceptional Cases

```ruby
class Recording::Copier
  def copy
    Event.suppress do
      # Events won't fire during copy
      @destination.record(@source.recordable)
    end
  end
end
```

## When to Use Callbacks vs Factories

| Use Callbacks | Use Factories |
|---------------|---------------|
| Simple, orthogonal concerns | Complex multi-step creation |
| Always happens on create | Conditional/configurable logic |
| Auditing, associated records | Business workflow orchestration |

## Caveats

- Reset happens automatically per request; in jobs, set what you need explicitly (e.g. `Current.set(person: event.creator) { ... }`) — job code must not assume request context exists.
- Keep the attribute list small: identity and request metadata, not a grab-bag of globals.
