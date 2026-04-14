# Architecture — core-web-api

Rails 8.1 **API-only**, multi-tenant by subdomain via `acts_as_tenant`. PostgreSQL, Solid Queue, JWT, Blueprinter, Pagy, Ransack, Discard.

---

## Business modules

| Module | Purpose | Key models |
|---|---|---|
| `Core` | Back-office foundation | `Customer`, `User`, `Subscription`, `UserSession`, `Audit` |
| `Customer` | Tenant users/configs | `Customer::User`, `Customer::WhatsappConfig` |
| `Katalog` | Unified catalog | `Item`, `Category`, `ItemVariant`, `ItemCategory` |
| `Kalendar` | Bookings and scheduling | `Booking`, `Resource`, `ResourceType`, `Availability` |
| `Kathy` | Chatbot/FAQ assistant | `Faq`, `Chat`, `ChatMessage`, `ToolCall`, `Model` |
| `Kommerce` | Orders and payment | `Order`, `OrderItem` |

**Key distinction**: `Core::User` = admin/back-office user; `Customer::User` = tenant-side user. `Current.user` is **always** `Customer::User` (there is no web path that would set `Core::User`).

---

## API audiences

APIs are versioned and namespaced by audience: `/api/v1/<audience>/<module>/...`

| Audience | Consumer | Auth | Tenant resolution |
|---|---|---|---|
| `admin` | Back-office panel | JWT | Subdomain |
| `customer` | Tenant app | JWT | Subdomain |
| `kathy` | WhatsApp bot | **No JWT** | **By `phone_number_id`** (WhatsApp webhook) |

---

## Base controllers

- **`ApplicationController`** (root) inherits from `ActionController::API`. Includes cross-cutting concerns:
  - `Attachable` — files
  - `CurrentContextable` — JWT → `Current.user`
  - `ExceptionsHandler` — unified errors
  - `Pagination` — Pagy helpers
  - `Tenantable` — subdomain → `ActsAsTenant.current_tenant`
- **All `Api::V1::*` controllers inherit directly from `ApplicationController`**. There is NO `Api::V1::Customer::BaseController` or `Api::V1::Admin::BaseController` — do not create them, do not assume they exist.
- **`Api::V1::Kathy::BaseController`** is the only audience-specific base controller: `skip_before_action :set_current_user, :set_tenant` + uses `set_tenant_by_phone_number`.

---

## Multi-tenancy

**Resolution** (`app/controllers/concerns/tenantable.rb`):
- Subdomain extracted from `request.origin || request.url`.
- `Core::Customer.find_by(subdomain: ...)` → `set_current_tenant(core_customer)`.
- Kathy: `Customer::WhatsappConfig.active.find_by!(phone_number_id: ...)`.

**Scoped models**:
```ruby
acts_as_tenant(:core_customer, class_name: "Core::Customer", foreign_key: :core_customer_id)
```
Queries are scoped automatically (`Katalog::Item.all` only returns the current tenant's records). **Never manually duplicate the scope**.

**Local dev**: `/etc/hosts` → `127.0.0.1 mycompany.local`, access via `http://mycompany.local:3000`.

---

## Authentication

- **JWT only** (no sessions or cookies).
- Header: `Authorization: Bearer <token>`.
- `CurrentContextable#set_current_user`:
  - `JWT.decode` with `ENV["JWT_SECRET_KEY"]` and `HS256`.
  - `Current.user = Customer::User.find_by!(id: decoded_jwt.first["id"])`.
  - Returns **401** on `JWT::DecodeError` or `ActiveRecord::RecordNotFound`.
- `Current` (`app/models/current.rb`): `ActiveSupport::CurrentAttributes` with `attribute :user, :customer`.

---

## Routes

Main `config/routes.rb` uses `draw()` to load each module:
```ruby
draw("api/admin/core")
draw("api/customer/kalendar")
draw("api/customer/katalog")
draw("api/customer/kathy")
draw("api/kathy/webhooks")
# etc.
```

**Rule**: each module/audience has its own file in `config/routes/api/<audience>/<module>.rb`. Don't mix.

---

## Serialization, pagination, search

- **Blueprinter** (`app/serializers/<module>/<resource>_serializer.rb`, **plural** name):
  ```ruby
  Katalog::ItemsSerializer.render(items)
  ```
- **Pagy** via the `Pagination` concern:
  ```ruby
  @pagy, records = custom_pagy(scope.ransack(params[:q]).result)
  ```
  Params: `pagination[page]` (default 1), `pagination[items]` (default 25). Headers are merged automatically in `after_action`.
- **Ransack** via the `Ransackable` model concern (`include Ransackable`) — exposes ALL columns and associations as searchable. Controllers: `Model.ransack(params[:q]).result`.

---

## Cross-cutting model concerns

Use whenever applicable:
- `Discardable` — soft deletes via `discard` gem.
- `Ransackable` — exposes attributes/associations to Ransack.

Typical pattern of a domain model:
```ruby
class Katalog::Item < ApplicationRecord
  include Discardable
  include Ransackable
  acts_as_tenant(:core_customer, class_name: "Core::Customer", foreign_key: :core_customer_id)
  # ...
end
```

---

## Code location map

```
app/
├── models/<module>/<resource>.rb
├── controllers/api/v1/<audience>/<module>/<resources>_controller.rb
├── serializers/<module>/<resources>_serializer.rb              # plural
├── recorders/<module>/<resources>/{create,update,destroy}_recorder.rb
├── services/<module>/...                                        # Query, Action, LLM Tool
├── decorators/<module>/<resource>_decorator.rb
├── jobs/<module>/<resource>_job.rb
└── mailers/<module>/<resource>_mailer.rb

config/routes/api/<audience>/<module>.rb

spec/
├── models/<module>/<resource>_spec.rb
├── requests/api/v1/<audience>/<module>/<resources>_spec.rb
├── recorders/<module>/<resources>/{create,update,destroy}_recorder_spec.rb
├── services/<module>/...
├── jobs/<module>/<resource>_job_spec.rb
└── mailers/<module>/<resource>_mailer_spec.rb
```

**Naming**:
- Recorders by operation: `Kalendar::Bookings::CreateRecorder`.
- Serializers in plural: `Katalog::ItemsSerializer`.
- Routes loaded with `draw("api/<audience>/<module>")`.

---

## Decision principles

1. **Audience first**: before creating a controller, decide whether it's admin, customer, or kathy. Each has distinct semantics (kathy doesn't even have JWT).
2. **Module by domain**: `Kalendar::Booking` (bookings) vs `Kommerce::Order` (payment/order) — don't mix responsibilities.
3. **Tenant isolation is law**: every tenant-scoped domain model → `acts_as_tenant`. No exceptions except `Core::Customer` (the tenant itself).
4. **Concerns before custom code**: `Discardable`, `Ransackable` already solve common cases. Reuse before duplicating.
