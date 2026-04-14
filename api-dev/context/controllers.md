# Controllers — REST API by audience

Controllers are **thin**: they parse params, delegate to Recorders/Services, serialize, and respond. Zero business logic.

---

## Inheritance

**ALL `Api::V1::*` controllers inherit directly from `::ApplicationController`**. There is NO `Api::V1::Customer::BaseController` or `Api::V1::Admin::BaseController` — do not create them, do not assume they exist.

```ruby
module Api
  module V1
    module Customer
      module Katalog
        class CategoriesController < ::ApplicationController
          # ...
        end
      end
    end
  end
end
```

**Single exception**: `Api::V1::Kathy::BaseController` — for WhatsApp webhooks that don't have JWT or subdomain. `skip_before_action :set_current_user, :set_tenant` and uses `set_tenant_by_phone_number`.

---

## `ApplicationController` — what it already does for you

Inherits from `ActionController::API` and includes:
- `Attachable` — file helpers
- `CurrentContextable` — JWT → `Current.user = Customer::User`, 401 on failure
- `ExceptionsHandler` — centralizes error responses (see section)
- `Pagination` — Pagy helpers (`custom_pagy`)
- `Tenantable` — subdomain → `ActsAsTenant.current_tenant`

**Don't reinvent any of these** in the specific controller.

---

## Anatomy of a CRUD — example: `Katalog::CategoriesController`

```ruby
module Api
  module V1
    module Customer
      module Katalog
        class CategoriesController < ::ApplicationController
          def index
            data = ::Katalog::Category.active.ransack(params[:q])
            @pagy, categories = custom_pagy(data.result)

            render json: ::Katalog::CategoriesSerializer.render(categories)
          end

          def create
            recorder = ::Katalog::Categories::CreateRecorder.new(params: permitted_params)
            recorder.record

            render json: ::Katalog::CategoriesSerializer.render(recorder.katalog_category), status: :created
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
            recorder = ::Katalog::Categories::DestroyRecorder.new(katalog_category: katalog_category)
            recorder.record

            head :no_content
          end

          private

          def katalog_category
            @katalog_category ||= ::Katalog::Category.active.find(params[:id])
          end

          def permitted_params
            params.require(:katalog_category).permit(:name)
          end
        end
      end
    end
  end
end
```

### Observations
- **`.active`** is a scope from the `Discardable` concern (equivalent to `.kept`). Always use it on public queries — never expose discarded records.
- **`.ransack(params[:q])`** is mandatory for endpoints with filter/search. The model's `Ransackable` concern enables all columns.
- **`custom_pagy(data.result)`** for pagination. Headers (`X-Page`, `X-Items`, etc.) are merged automatically in `after_action`.
- **Memoized finder** (`@katalog_category ||= ...`) — avoids duplicate queries between `before_action` and the action.
- **`permitted_params`** ALWAYS with `params.require(:<resource>).permit(...)`. The key is `<module>_<resource>` snake_case.

---

## Patterns per action

### `index` — listing with search and pagination

```ruby
def index
  data = ::Katalog::Category.active.ransack(params[:q])
  @pagy, categories = custom_pagy(data.result)
  render json: ::Katalog::CategoriesSerializer.render(categories)
end
```

**With eager loading** (when the serializer includes associations):
```ruby
def index
  data = ::Kalendar::Booking.active
                            .includes(:booked_by, :created_by, kalendar_resource: [:kalendar_availabilities, :kalendar_resource_type])
                            .ransack(params[:q])
  @pagy, bookings = custom_pagy(data.result)
  render json: ::Kalendar::BookingsSerializer.render(bookings)
end
```

**Rule**: if the serializer needs associations, use `includes` to avoid N+1. Don't let the serializer trigger queries.

### `show` — finder + serializer

```ruby
def show
  render json: ::Kalendar::BookingsSerializer.render(kalendar_booking)
end
```

### `create` — recorder + `status: :created`

```ruby
def create
  recorder = ::Kalendar::Bookings::CreateRecorder.new(params: permitted_params)
  recorder.record

  render json: ::Kalendar::BookingsSerializer.render(recorder.kalendar_booking), status: :created
end
```

**Rule**: `status: :created` (201) in a successful create response. Default 200 doesn't apply here.

### `update` — recorder with resource + params

```ruby
def update
  recorder = ::Kalendar::Bookings::UpdateRecorder.new(
    params: permitted_params,
    kalendar_booking: kalendar_booking
  )
  recorder.record

  render json: ::Kalendar::BookingsSerializer.render(recorder.kalendar_booking)
end
```

### `destroy` — two variants

**No body** (204 No Content):
```ruby
def destroy
  ::Katalog::Categories::DestroyRecorder.new(katalog_category: katalog_category).record
  head :no_content
end
```

**With body** (when the client needs confirmation of the discarded record):
```ruby
def destroy
  recorder = ::Kalendar::Bookings::DestroyRecorder.new(kalendar_booking: kalendar_booking)
  recorder.record

  render json: ::Kalendar::BookingsSerializer.render(recorder.kalendar_booking)
end
```

**Rule**: use `head :no_content` by default. Only serialize if the frontend explicitly requires it.

### Custom actions (e.g., `stats`, `search`)

```ruby
def stats
  render json: ::Kalendar::Booking.stats
end
```

**Warning signal**: if a custom action has more than 3-4 lines of logic, extract it to a Query Service (`app/services/`).

---

## Error handling — **centralized**

**NEVER rescue in the controller**. The `ExceptionsHandler` concern already rescues:

| Exception | Status | Body |
|---|---|---|
| `ActiveRecord::RecordInvalid` | 422 | `{ error: record.errors.full_messages }` |
| `ActiveRecord::RecordNotDestroyed` | 422 | `{ error: record.errors.full_messages }` |
| `ActiveRecord::RecordNotFound` | 404 | `{ error: "Record not found" }` |
| `ActiveRecord::RecordNotUnique` | 422 | `{ error: "Record already exists" }` |
| Invalid JWT / user not found | 401 | (from `CurrentContextable`) |

**Rule**: if you need a custom error, first consider if it fits into the ones above. If you really need a new one, add it to the central `ExceptionsHandler` — don't rescue locally.

---

## Routes

Separate files in `config/routes/api/<audience>/<module>.rb`. Loaded from `config/routes.rb` with `draw()`.

### Standard structure — with `constraints(subdomain:)`

```ruby
# config/routes/api/customer/katalog.rb
Rails.application.routes.draw do
  constraints(subdomain: /.+/) do
    namespace :api do
      namespace :v1 do
        namespace :customer do
          namespace :katalog do
            resources :categories, except: :index do
              post :search, on: :collection, action: :index
            end
          end
        end
      end
    end
  end
end
```

### Critical observations

- **`constraints(subdomain: /.+/)`** mandatory for `admin` and `customer` audiences. Without this, the tenant isn't resolved.
- **Full `namespace` chain**: `api → v1 → <audience> → <module>`.
- **`resources`** with exceptions if applicable (`except: :index` because the index is handled via a custom route).
- **`post :search, on: :collection, action: :index`** is the pattern for **search via POST body** — allows sending complex filters that don't fit in querystring. Maps to the controller's `index` action.
- **Kathy (webhooks)** does NOT use `constraints(subdomain:)` — tenant is resolved by phone_number_id.

### Register the route in `config/routes.rb`

```ruby
Rails.application.routes.draw do
  draw("api/customer/katalog")   # ← add here
end
```

---

## Pagination (reminder)

Params: `pagination[page]` (default 1), `pagination[items]` (default 25).

```ruby
@pagy, records = custom_pagy(scope.ransack(params[:q]).result)
```

Automatic headers in response (via `after_action` in the `Pagination` concern):
- `X-Page`, `X-Items`, `X-Count`, `X-Pages`

---

## Serializers — Blueprinter

- Namespace with **plural** name: `Katalog::CategoriesSerializer`, `Kalendar::BookingsSerializer`.
- Render: `Serializer.render(obj_or_collection)`.
- Views: `Serializer.render(obj, view: :detailed)` (if the serializer defines `view :detailed`).

**Rule**: one serializer per resource. Do not use `as_json` or manual `to_json`.

---

## Permitted params

The key is `<module>_<resource>` in snake_case:

```ruby
params.require(:katalog_category).permit(:name)
params.require(:kalendar_booking).permit(:start_at, :end_at, :status, :notes, :kalendar_resource_id, :booked_by_id)
```

**Rule**: only permit attributes the user can set. `core_customer_id` is never in `permit` — it comes from the tenant automatically via `acts_as_tenant`.

---

## Testing — two approaches

Request specs in `spec/requests/api/v1/<audience>/<module>/<resources>_spec.rb` with `type: :request`.

### Unit approach (default, fast, no DB)

Uses `allow_any_instance_of` to bypass auth/tenant + `instance_double` for models. **No** `create(:factory)`.

```ruby
RSpec.describe "API V1 Customer Katalog Categories", type: :request do
  before do
    host! "test.example.com"
    allow_any_instance_of(::ApplicationController).to receive(:set_current_user).and_return(true)
    allow_any_instance_of(::ApplicationController).to receive(:set_tenant).and_return(true)
  end

  describe "POST /api/v1/customer/katalog/categories" do
    let(:category) { instance_double(::Katalog::Category, id: 1) }
    let(:recorder) { instance_double(::Katalog::Categories::CreateRecorder, record: true, katalog_category: category) }

    it "delegates to recorder and returns serialized JSON" do
      expect(::Katalog::Categories::CreateRecorder).to receive(:new)
        .with(params: ActionController::Parameters.new(name: "Test").permit!)
        .and_return(recorder)
      expect(::Katalog::CategoriesSerializer).to receive(:render).with(category).and_return('{"id":1}')

      post "/api/v1/customer/katalog/categories", params: { katalog_category: { name: "Test" } }

      expect(response).to have_http_status(:created)
      expect(response.body).to eq('{"id":1}')
    end
  end

  describe "GET /api/v1/customer/katalog/categories/:id (not found)" do
    it "returns 404 when record missing" do
      allow(::Katalog::Category).to receive_message_chain(:active, :find).and_raise(ActiveRecord::RecordNotFound)
      get "/api/v1/customer/katalog/categories/999"
      expect(response).to have_http_status(:not_found)
    end
  end
end
```

### Integration approach (exception, requires justification)

Uses `include_context 'api_authentication'` + `create(:factory)`. **Reserved** for critical end-to-end flows (auth, multi-step, complex side effects). Every usage must have a comment explaining why it couldn't be done with mocks.

```ruby
RSpec.describe "API V1 Customer Katalog Categories integration", type: :request do
  include_context 'api_authentication'   # customer_user + authorization + tenant

  describe "POST /api/v1/customer/katalog/categories" do
    # NOTE: integration test justified — validates tenant scoping end-to-end
    it "creates a category scoped to the current tenant" do
      post "/api/v1/customer/katalog/categories",
           headers: { Authorization: authorization },
           params: { katalog_category: { name: "Integration" } }

      expect(response).to have_http_status(:created)
      category = ::Katalog::Category.last
      expect(category.core_customer_id).to eq(core_customer.id)
    end
  end
end
```

---

## When an action deviates from the standard CRUD pattern

| Situation | Solution |
|---|---|
| Search with complex filters that don't fit in GET | `post :search, on: :collection, action: :index` |
| Stats/aggregates | Custom method in controller + scope in model (`Model.stats`) |
| Multi-resource operation (e.g., create booking + payment) | Action Service invoked from the controller |
| Actions on a resource (e.g., confirm booking) | Member route: `patch :confirm, on: :member` + method delegating to Recorder |

---

## Checklist for creating a new Controller

- [ ] Inherits directly from `::ApplicationController` (no audience BaseController).
- [ ] Location: `app/controllers/api/v1/<audience>/<module>/<resources>_controller.rb`.
- [ ] Each CRUD action delegates to a Recorder — zero business logic in the controller.
- [ ] `index` uses `.active.ransack(params[:q])` + `custom_pagy`.
- [ ] Eager loading with `includes` if the serializer touches associations.
- [ ] `create` responds with `status: :created`.
- [ ] `destroy` responds with `head :no_content` unless the frontend requires a body.
- [ ] `permitted_params` uses `require(:<module>_<resource>).permit(...)` — without `core_customer_id`.
- [ ] Memoized finder (`@resource ||= Model.active.find(params[:id])`).
- [ ] **No `rescue`** — `ExceptionsHandler` handles it.
- [ ] Serializer with plural name: `<Module>::<Resources>Serializer`.
- [ ] Route added in `config/routes/api/<audience>/<module>.rb` with `constraints(subdomain: /.+/)`.
- [ ] Route registered in `config/routes.rb` with `draw("api/<audience>/<module>")`.
- [ ] Request spec with Unit approach (mocks) — Integration only if justified.
