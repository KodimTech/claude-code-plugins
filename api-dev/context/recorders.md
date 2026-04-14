# Recorders — CRUD + Event Pattern

The **Recorder** is the most unique pattern of core-web-api. It replaces the persistence logic that would normally live in the controller or the model with a dedicated class that performs **one** operation and emits **one** event.

---

## Single responsibility

A Recorder:
- Receives `params` and optionally the existing resource.
- Executes **one** ActiveRecord operation (`create!`, `update!`, `discard!`).
- Emits **one** event per operation (array of symbols).
- Returns the persisted resource.

A Recorder does **not**:
- Orchestrate multiple operations (that's an `Action Service`).
- Call external APIs or send emails (that's a `Job` or `Service`).
- Validate beyond the model's `validates` (that's the model's responsibility).
- Make authorization decisions (that's the controller's or Pundit-equivalent).

---

## Hard rules

1. **One operation = one recorder**: `CreateRecorder`, `UpdateRecorder`, `DestroyRecorder`. Never an "upsert recorder".
2. **One event per operation**: `:katalog_category_created`, not a mixed list.
3. **`events` is a public mutable array** via `attr_reader`. Consumers read, don't write.
4. **`record` returns the model** (even though it's also exposed as an `attr_reader`).
5. **Destroy uses `discard!`**, never `destroy!` (the repo is soft-delete first).
6. **Do not catch errors** — propagate them. If `create!` fails, the event is NOT added and the exception bubbles up.
7. **No cross-cutting concerns** except for calculations (e.g., `Concerns::TotalCalculator`). The recorder is a simple black box.

---

## Anatomy — real example: `Katalog::Categories::CreateRecorder`

```ruby
module Katalog
  module Categories
    class CreateRecorder
      attr_reader :events, :params, :katalog_category

      def initialize(params:)
        @params = params
        @events = []
      end

      def record
        create_category
        katalog_category
      end

      private

      def create_category
        @katalog_category = ::Katalog::Category.create!(params)
        events << :katalog_category_created
      end
    end
  end
end
```

**Observations**:
- `attr_reader` exposes `events`, `params`, and the model (`katalog_category`).
- `@events = []` is initialized in the constructor.
- The private `create_category` method is atomic: creates + emits event. If `create!` blows up, the event is NOT added.
- Uses `::Katalog::Category` with `::` to reference the model unambiguously.
- The instance variable name uses **snake_case with module prefix**: `katalog_category`, not just `category`.

---

## The 3 Recorder types

### Create — only `params:`
```ruby
class CreateRecorder
  attr_reader :events, :params, :katalog_category

  def initialize(params:)
    @params = params
    @events = []
  end

  def record
    create_category
    katalog_category
  end

  private

  def create_category
    @katalog_category = ::Katalog::Category.create!(params)
    events << :katalog_category_created
  end
end
```

### Update — `params:` + existing resource
```ruby
class UpdateRecorder
  attr_reader :events, :params, :katalog_category

  def initialize(params:, katalog_category:)
    @params = params
    @katalog_category = katalog_category
    @events = []
  end

  def record
    update_category
    katalog_category
  end

  private

  def update_category
    katalog_category.update!(params)
    events << :katalog_category_updated
  end
end
```

### Destroy — only existing resource
```ruby
class DestroyRecorder
  attr_reader :events, :katalog_category

  def initialize(katalog_category:)
    @katalog_category = katalog_category
    @events = []
  end

  def record
    destroy_category
    katalog_category
  end

  private

  def destroy_category
    katalog_category.discard!    # ← discard, NOT destroy
    events << :katalog_category_destroyed
  end
end
```

---

## Naming conventions

| Element | Format | Example |
|---|---|---|
| Resource namespace | **plural** | `Katalog::Categories::` |
| Recorder class | `{Create\|Update\|Destroy}Recorder` | `CreateRecorder` |
| Model variable | snake_case with **module prefix** | `katalog_category` (not `category`) |
| Event | `:<resource_snake>_<operation>` | `:katalog_category_created` |
| Model keyword arg | same as the variable | `katalog_category:` |

**Rule**: the module prefix avoids collisions when a recorder from one module works with models from another (e.g., a Recorder in `Kommerce` that indirectly creates a `Kalendar::Booking`).

---

## Usage from the controller

The controller **only** orchestrates. No business logic:

```ruby
def create
  recorder = ::Katalog::Categories::CreateRecorder.new(params: permitted_params)
  recorder.record

  render json: ::Katalog::CategoriesSerializer.render(recorder.katalog_category),
         status: :created
end

def update
  recorder = ::Katalog::Categories::UpdateRecorder.new(
    params: permitted_params,
    katalog_category: katalog_category
  )
  recorder.record

  render json: ::Katalog::CategoriesSerializer.render(recorder.katalog_category)
end

def destroy
  ::Katalog::Categories::DestroyRecorder.new(katalog_category: katalog_category).record
  head :no_content
end
```

**Errors**: `ActiveRecord::RecordInvalid` and `ActiveRecord::RecordNotDestroyed` bubble up to `ExceptionsHandler` → 422. The controller does NOT `rescue` — that's centralized.

---

## Testing — mandatory pattern

Two specs per Recorder: happy path and error path. Use `instance_double` to isolate from the real model.

```ruby
require "rails_helper"

RSpec.describe Katalog::Categories::CreateRecorder do
  subject(:recorder) { described_class.new(params: params) }

  let(:params) { { name: "New Category", core_customer_id: 1 } }
  let(:category) { instance_double(::Katalog::Category) }

  describe "#record" do
    it "creates the category, returns the record and adds the event" do
      expect(::Katalog::Category).to receive(:create!).with(params).and_return(category)

      expect { expect(recorder.record).to eq(category) }
        .to change { recorder.events }.from([]).to([:katalog_category_created])
    end

    it "propagates the error and does not add events when create! fails" do
      error = ActiveRecord::RecordInvalid.new(::Katalog::Category.new)
      expect(::Katalog::Category).to receive(:create!).with(params).and_raise(error)

      expect { recorder.record }.to raise_error(error.class)
      expect(recorder.events).to be_empty
    end
  end
end
```

**Key spec points**:
- `instance_double(::Katalog::Category)` isolates from the real model → fast test, no DB.
- Verifies **2 things atomically**: return value + change in `events`.
- Error test verifies **both**: the exception bubbles up + `events` remains empty (no "half-committed state").
- `::Model` with double colon to avoid any weird relative resolution.

**Specs per type**:
- `Create`: mock `Model.create!`
- `Update`: pass `instance_double` of the model to the constructor, mock `model.update!`
- `Destroy`: pass `instance_double`, mock `model.discard!` (NOT `destroy!`)

---

## Shared concerns (exceptional use)

Only when multiple recorders need the **same calculation**. Real example: `Kalendar::Bookings::Concerns::TotalCalculator`:

```ruby
module Kalendar
  module Bookings
    class CreateRecorder
      include Concerns::TotalCalculator   # ← only valid case

      def create_booking
        @kalendar_booking = ::Kalendar::Booking.new(params)
        set_total                          # method from the concern
        @kalendar_booking.save!
        events << :kalendar_booking_created
      end
    end
  end
end
```

**Rule**: a concern only if the logic is duplicated across ≥2 recorders of the same resource. Don't create concerns preemptively.

---

## When NOT to use a Recorder

- **Query**: I need to read data without mutating → `Query Service` (see `services.md`).
- **Orchestration**: I need to create multiple models + send email + enqueue a job → `Action Service` that internally calls N recorders.
- **Complex conditional logic**: "create A if B, else update C" → `Action Service`.
- **External call**: Stripe, WhatsApp, etc. → `Service` + `Job`.

**Rule of thumb**: if you're tempted to put an `if` that changes flow inside `record`, it's not a recorder — it's a service.

---

## Checklist for creating a new Recorder

- [ ] Namespace: `<Module>::<Resources>::<Operation>Recorder`
- [ ] `attr_reader :events, :params, :<resource_snake>` (as per type)
- [ ] `@events = []` in the constructor
- [ ] `#record` is public, delegates to a private method, returns the model
- [ ] Private method does **one** AR operation + pushes the event
- [ ] Destroy uses `discard!` (not `destroy!`)
- [ ] Event in `:<resource_snake>_<operation>` format
- [ ] Reference to the model with `::` prefix
- [ ] Spec with happy path + error path, using `instance_double`
- [ ] Error test verifies `events` remains empty
