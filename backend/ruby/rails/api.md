# Ruby on Rails 7.2 — API Production Skill

## Role

You are a **senior Rails engineer** on Rails 7.2 + Ruby 3.3+. You build API-only Rails apps that are fast, secure, and follow the "convention over configuration" philosophy — but you know exactly which conventions to override for production.

**Version baseline**: Rails 7.2 · Ruby 3.3 · Devise + JWT or Rodauth · ActiveRecord + PostgreSQL

---

## File Structure (Rails API — domain-aware)

```
app/
├── controllers/
│   ├── api/
│   │   └── v1/
│   │       ├── base_controller.rb      ← auth + error handling
│   │       └── orders/
│   │           └── orders_controller.rb
│   └── concerns/
│       └── api_error_handler.rb
│
├── models/
│   ├── order.rb
│   ├── order_item.rb
│   └── concerns/
│       └── user_scoped.rb
│
├── serializers/                        ← use Alba or ActiveModelSerializers
│   └── order_serializer.rb
│
├── services/                           ← business logic
│   └── orders/
│       ├── create_order_service.rb
│       └── cancel_order_service.rb
│
├── policies/                           ← Pundit authorization
│   └── order_policy.rb
│
└── queries/                            ← query objects
    └── orders_query.rb
```

---

## Routes — Versioned

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :orders, only: [:index, :show, :create, :update, :destroy] do
        member do
          patch :cancel
        end
      end
      resources :products, only: [:index, :show]
    end
  end

  get '/health', to: proc { [200, {}, [{ status: 'ok' }.to_json]] }
end
```

---

## Base Controller

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ApplicationController
      include Pundit::Authorization
      include ApiErrorHandler

      before_action :authenticate_user!
      after_action :verify_authorized, except: :index
      after_action :verify_policy_scoped, only: :index

      rescue_from Pundit::NotAuthorizedError, with: :render_forbidden
      rescue_from ActiveRecord::RecordNotFound, with: :render_not_found

      private

      def current_user
        @current_user ||= authenticate_jwt
      end

      def authenticate_user!
        render_unauthorized unless current_user
      end

      def render_unauthorized
        render json: { message: 'Unauthorized' }, status: :unauthorized
      end

      def render_forbidden
        render json: { message: 'Forbidden' }, status: :forbidden
      end

      def render_not_found
        render json: { message: 'Not found' }, status: :not_found
      end
    end
  end
end
```

---

## Controller — Thin

```ruby
# app/controllers/api/v1/orders/orders_controller.rb
module Api
  module V1
    module Orders
      class OrdersController < BaseController

        def index
          orders = policy_scope(Order).then { |q| OrdersQuery.new(q, params).call }
                                      .page(params[:page]).per(params[:per_page] || 15)
          render json: OrderSerializer.new(orders).serializable_hash
        end

        def show
          order = find_order
          authorize order
          render json: OrderSerializer.new(order).serializable_hash
        end

        def create
          authorize Order
          result = Orders::CreateOrderService.call(current_user, order_params)

          if result.success?
            render json: OrderSerializer.new(result.order).serializable_hash,
                   status: :created
          else
            render json: { message: 'Validation failed', errors: result.errors },
                   status: :unprocessable_entity
          end
        end

        def destroy
          order = find_order
          authorize order
          order.destroy!
          head :no_content
        end

        private

        def find_order
          policy_scope(Order).find(params[:id])  # scoped to user
        end

        def order_params
          params.require(:order).permit(
            :customer_id, :payment_method, :currency, :notes,
            items: [:product_id, :quantity, :unit_price]
          )
        end
      end
    end
  end
end
```

---

## Model — ActiveRecord Best Practices

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  belongs_to :user
  belongs_to :customer
  has_many :items, class_name: 'OrderItem', dependent: :destroy

  enum :status, {
    pending:   'pending',
    confirmed: 'confirmed',
    shipped:   'shipped',
    delivered: 'delivered',
    cancelled: 'cancelled'
  }, prefix: true

  validates :customer_id, presence: true
  validates :currency,    presence: true, inclusion: { in: %w[USD EUR GBP] }
  validates :total,       presence: true, numericality: { greater_than: 0 }

  scope :for_user, ->(user) { where(user:) }
  scope :by_status, ->(status) { status.present? ? where(status:) : all }
  scope :recent, -> { order(created_at: :desc) }
  scope :with_associations, -> { includes(:customer, items: :product) }

  def can_cancel?
    pending? || confirmed?
  end
end

# Avoid N+1 — always specify includes in scopes
# Order.for_user(user).with_associations.page(1)
```

---

## Pundit Policy — Authorization

```ruby
# app/policies/order_policy.rb
class OrderPolicy < ApplicationPolicy

  def index?  = true   # scope handles data filtering
  def show?   = owner?
  def create? = user.active?
  def update? = owner? && record.pending?
  def destroy? = owner? && record.can_cancel?

  class Scope < ApplicationPolicy::Scope
    def resolve
      scope.for_user(user)  # automatically scope to current user
    end
  end

  private

  def owner?
    record.user_id == user.id
  end
end
```

---

## Service Object Pattern

```ruby
# app/services/orders/create_order_service.rb
module Orders
  class CreateOrderService
    Result = Struct.new(:success?, :order, :errors)

    def self.call(user, params)
      new(user, params).call
    end

    def initialize(user, params)
      @user   = user
      @params = params
    end

    def call
      ActiveRecord::Base.transaction do
        order = Order.new(
          user:           @user,
          customer_id:    @params[:customer_id],
          payment_method: @params[:payment_method],
          currency:       @params[:currency] || 'USD',
          notes:          @params[:notes],
          status:         :pending,
          total:          calculate_total(@params[:items]),
        )

        @params[:items]&.each do |item|
          order.items.build(
            product_id: item[:product_id],
            quantity:   item[:quantity].to_i,
            unit_price: item[:unit_price].to_f,
          )
        end

        if order.save
          Result.new(true, order, nil)
        else
          Result.new(false, nil, order.errors.to_hash)
        end
      end
    end

    private

    def calculate_total(items)
      (items || []).sum { |i| i[:quantity].to_f * i[:unit_price].to_f }
    end
  end
end
```

---

## Serializer — Alba (fast, explicit)

```ruby
# app/serializers/order_serializer.rb
class OrderSerializer
  include Alba::Resource

  attributes :id, :status, :total, :currency, :created_at

  attribute :status_label do |order|
    order.status.humanize
  end

  one :customer, resource: CustomerSerializer
  many :items, resource: OrderItemSerializer, if: ->(order) { order.association(:items).loaded? }

  # REJECT: as_json — exposes all fields including user_id, internal fields
end
```

---

## Common Anti-Patterns

```ruby
# REJECT: Logic in controller
def create
  order = Order.new(params)
  order.calculate_total
  order.send_confirmation_email
  order.update_inventory
  order.save
end
# USE: CreateOrderService

# REJECT: No authorization (Pundit policy missing)
def show
  @order = Order.find(params[:id])  # any order, any user — IDOR
end

# REJECT: serialize with as_json
render json: @order.as_json  # exposes all DB columns

# REJECT: N+1 in controller
def index
  @orders = Order.for_user(current_user)  # then template calls order.customer.name for each
end
# USE: .includes(:customer, :items)

# REJECT: Raw SQL with string interpolation
Order.where("status = '#{params[:status]}'")  # SQL injection
# USE: Order.where(status: params[:status])

# REJECT: Not verifying Pundit
# after_action :verify_authorized is in BaseController — don't skip it
```

---

## N+1 — Bullet Gem in Development

```ruby
# Gemfile (development)
gem 'bullet'

# config/environments/development.rb
config.after_initialize do
  Bullet.enable        = true
  Bullet.alert         = true
  Bullet.rails_logger  = true
  Bullet.add_footer    = true
end
```

---

## application.rb — Production Settings

```ruby
config.force_ssl = true
config.log_level = :info
config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }

# API-only app — no sessions, no CSRF
config.api_only = true
# When api_only: true, SessionMiddleware and CSRF are already removed
```

---

## Deployment Gates

- [ ] `rails credentials:show` — all secrets in credentials, not hardcoded
- [ ] `bundle exec rails db:migrate:status` — all migrations up
- [ ] `bundle exec rubocop --parallel` — no offenses
- [ ] `bundle exec rspec` — all passing
- [ ] `Bullet` has zero warnings in staging
- [ ] `config.api_only = true` set
- [ ] `config.force_ssl = true` in production
- [ ] Pundit `verify_authorized` and `verify_policy_scoped` active on BaseController
- [ ] All serializers use Alba or explicit serializer — no `as_json`
- [ ] Rate limiting configured (Rack::Attack)
