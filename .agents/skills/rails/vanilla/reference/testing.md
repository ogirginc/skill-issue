# Testing: Minitest + Fixtures

The 37signals testing stack is deliberately boring: Minitest, fixtures, and mostly model tests. No RSpec, no FactoryBot, minimal mocking.

## Fixtures Over Factories

Fixtures form one shared "known universe" loaded once per test run — fast (inserted in bulk, wrapped in transactions) and forcing you to keep a coherent, realistic dataset instead of building bespoke object graphs per test.

```yaml
# test/fixtures/recordings.yml
launch_message:
  bucket: product
  creator: david
  recordable: launch (Message)

# test/fixtures/messages.yml
launch:
  subject: "We're live!"
```

```ruby
class RecordingTest < ActiveSupport::TestCase
  test "copying points the new recording at the same recordable" do
    recording = recordings(:launch_message)
    destination = buckets(:marketing)

    copy = recording.copy_to(destination)

    assert_equal recording.recordable, copy.recordable
    assert_equal destination, copy.bucket
  end
end
```

Guidelines:

- Name fixtures by role (`david`, `launch_message`), not `user_1`.
- A handful of rich, interconnected fixtures beats dozens of minimal ones.
- Need a variation? Start from a fixture and update it in the test, or add a new fixture if the shape recurs.

## Test Distribution

- **Model tests carry the weight.** Rich domain models mean the interesting logic is directly callable — test `recording.copy_to(destination)`, not the HTTP layer around it.
- **Controller/integration tests stay thin**: right status, right redirect, right authorization. One or two per controller action.
- **System tests sparingly** — only for flows where JS behavior itself is the feature.

```ruby
class Recordings::CopiesControllerTest < ActionDispatch::IntegrationTest
  test "copying a recording" do
    sign_in_as people(:david) # from the Rails 8 authentication generator

    assert_difference -> { buckets(:marketing).recordings.count } do
      post recording_copies_url(recordings(:launch_message)),
        params: { bucket_id: buckets(:marketing).id }
    end

    assert_redirected_to recording_url(Recording.last)
  end
end
```

## Testing Broadcasts and Frames

turbo-rails adds these assertions to every test case. Refresh broadcasts go out from a background job, so run the jobs inside the block:

```ruby
class MessageTest < ActiveSupport::TestCase
  test "a new message refreshes pages showing its bucket" do
    bucket = buckets(:product)

    assert_turbo_stream_broadcasts bucket do
      perform_enqueued_jobs { bucket.messages.create!(subject: "Hello") }
    end
  end
end
```

In integration tests, check frames in the rendered page. IDs are positional; a record plus a prefix becomes its `dom_id`:

```ruby
get bucket_url(buckets(:product))

assert_turbo_frame :notifications, loading: "lazy"
assert_turbo_frame messages(:launch), :preview do
  assert_select "h2", text: messages(:launch).subject
end
```

## Jobs and External HTTP

Adapted from Fizzy's `test_helper.rb`:

```ruby
# test/test_helper.rb
require "webmock/minitest"
require "vcr"

VCR.configure do |config|
  config.cassette_library_dir = "test/vcr_cassettes"
  config.hook_into :webmock
  config.filter_sensitive_data("<API_KEY>") { Rails.application.credentials.api_key }
end

class ActiveSupport::TestCase
  include ActiveJob::TestHelper

  # Jobs must restore their own account (AccountTenanted in current-attributes.md),
  # not inherit it from the test's setup.
  def perform_enqueued_jobs(...)
    saved_account = Current.account
    Current.account = nil
    super
  ensure
    Current.account = saved_account
  end
end
```

- Check that a job was queued with `assert_enqueued_with(job: Recording::IncinerationJob)`; run it with `perform_enqueued_jobs`.
- Stub one-off HTTP calls with WebMock (`stub_request(:post, url).to_return(status: 200)`).
- For a third-party API, record real responses once with VCR (`VCR.use_cassette("openai/summary") { ... }`) and replay them on every run.

## Testing Code That Uses Current

Set `Current` explicitly in setup rather than stubbing:

```ruby
class RecordingTest < ActiveSupport::TestCase
  setup do
    Current.person = people(:david)
  end
end
```

(`CurrentAttributes` resets automatically between tests.)

## Principles

- Test behavior through public APIs; don't assert on internals or callbacks directly.
- Prefer real objects to mocks — with fixtures, collaborators are cheap.
- `parallelize(workers: :number_of_processors)` is on by default; keep tests independent of execution order.
