# CurrentAttributes + Callbacks

Thread-safe request context combined with callbacks for orthogonal concerns.

## Current Class

Start sign-in from Rails 8's authentication generator (`bin/rails generate authentication`) rather than hand-rolling it. It creates a `Session` model, an `Authentication` concern (`resume_session`, `allow_unauthenticated_access`), a rate-limited `SessionsController`, and a `Current` with `attribute :session`. The generator names the model `User`; examples here call it `Person`.

Extend that `Current` with the tenant and request details. Each setter fills in the next attribute, the way Fizzy's `Current` does. Keep `person` a real attribute (instead of the generated `delegate :user, to: :session`) so jobs and tests can set it without a session:

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :person, :account
  attribute :request_id, :user_agent, :ip_address

  def session=(session)
    super
    self.person = session&.person
  end

  def person=(person)
    super
    self.account = person&.account
  end
end
```

## Setting Current (in ApplicationController)

The generated `Authentication` concern sets `Current.session` from the session cookie; add the request details yourself:

```ruby
class ApplicationController < ActionController::Base
  include Authentication

  before_action :set_current_request_details

  private
    def set_current_request_details
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
  belongs_to :creator, class_name: "Person", default: -> { Current.person }
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
    has_many :events, as: :eventable, dependent: :destroy
    after_create :track_created
  end

  def track_event(action, creator: Current.person, **particulars)
    events.create!(
      action: action,
      creator: creator,
      request_id: Current.request_id,
      user_agent: Current.user_agent,
      ip_address: Current.ip_address,
      particulars: particulars
    )
  end

  private
    def track_created
      track_event :created
    end
end
```

`Eventable` needs a polymorphic `eventable` reference and a JSON `particulars` column on `events`, the same shape as Fizzy's `track_event(action, creator: Current.user, **particulars)`. One events table then serves every model, and new details (`track_event :moved, from_bucket_id: ...`) need no new columns. The `creator:` keyword lets code outside a request, like a job, name the actor explicitly.

## Suppress for Exceptional Cases

```ruby
class Recording::Copier
  def copy
    Event.suppress do
      # Events won't fire during copy
      @destination.record(@recording.recordable)
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

## Carrying the Account into Jobs

Jobs run outside the request, so `Current` starts empty. Instead of setting the tenant in every `perform`, capture it when the job is enqueued and restore it around execution. Adapted from Fizzy's `AccountTenanted`:

```ruby
# app/jobs/concerns/account_tenanted.rb
module AccountTenanted
  extend ActiveSupport::Concern

  prepended do
    attr_reader :account
    around_perform :with_account_context
  end

  def initialize(...)
    super
    @account = Current.account
  end

  def serialize
    super.merge({ "account" => @account&.to_gid&.to_s })
  end

  def deserialize(job_data)
    super
    @account_gid = job_data["account"]
  end

  private
    def with_account_context(&block)
      resolve_account!

      if account.present?
        Current.set(account: account, &block)
      else
        yield
      end
    end

    # Resolved here rather than in deserialize, so a deleted account raises
    # DeserializationError inside execution, where discard_on can handle it.
    def resolve_account!
      if @account_gid
        @account = GlobalID::Locator.locate(@account_gid)
      end
    rescue ActiveRecord::RecordNotFound
      raise ActiveJob::DeserializationError
    end
end

# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  prepend AccountTenanted
end
```

Jobs that Rails enqueues for you don't inherit from `ApplicationJob`, so prepend the module to them too; otherwise `deliver_later` and `broadcast_*_later` lose the tenant:

```ruby
# config/initializers/active_job.rb
ActiveSupport.on_load(:action_mailer) do
  ActionMailer::MailDeliveryJob.prepend AccountTenanted
end

Rails.application.config.after_initialize do
  Turbo::Streams::ActionBroadcastJob.prepend AccountTenanted
  Turbo::Streams::BroadcastJob.prepend AccountTenanted
  Turbo::Streams::BroadcastStreamJob.prepend AccountTenanted
end
```

Only the account is restored. Pass anything else a job needs explicitly: `track_event :incinerated, creator: event.creator`, or `Current.set(person: event.creator) { ... }` around code that reads `Current.person`.

## Caveats

- Reset happens automatically per request. Job code must not assume request context exists; it gets only what you restore (the account, above) or set explicitly.
- Keep the attribute list small: identity and request metadata, not a grab-bag of globals.
