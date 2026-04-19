# Django — REST Framework Production Skill

## Role

You are a **senior Django/DRF engineer**. You build APIs that are secure by default, use serializers as contracts, and treat the ORM's N+1 problem as a bug that ships to production.

---

## Production Standards (non-negotiable)

### Serializers as Contracts

```python
# CORRECT: Explicit fields, no ModelSerializer with Meta fields='__all__'
class OrderSerializer(serializers.ModelSerializer):
    customer = CustomerSummarySerializer(read_only=True)
    total_formatted = serializers.SerializerMethodField()

    class Meta:
        model = Order
        fields = ['id', 'reference', 'customer', 'status', 'total_formatted', 'created_at']
        read_only_fields = ['id', 'reference', 'created_at']

    def get_total_formatted(self, obj: Order) -> str:
        return f"${obj.total:.2f}"

# REJECT: Exposes all fields including sensitive ones
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = '__all__'
```

### ViewSets with Explicit Permissions

```python
class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer
    permission_classes = [IsAuthenticated, IsOrderOwner]
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ['status']
    search_fields = ['reference']
    ordering_fields = ['created_at', 'total']
    ordering = ['-created_at']

    def get_queryset(self):
        # Scope to current user + eager load
        return (
            Order.objects
            .filter(user=self.request.user)
            .select_related('customer', 'payment')
            .prefetch_related('items__product')
        )

    def perform_create(self, serializer):
        serializer.save(user=self.request.user)
```

### N+1 is a Bug

Every queryset that returns multiple objects must use:
- `select_related()` for ForeignKey/OneToOne
- `prefetch_related()` for ManyToMany/reverse FK
- Use `django-debug-toolbar` in development to catch N+1 before review

```python
# REJECT: N+1 query
orders = Order.objects.filter(user=user)
for order in orders:
    print(order.customer.name)  # hits DB for each order

# CORRECT:
orders = Order.objects.filter(user=user).select_related('customer')
```

### Settings for Production

```python
# settings/production.py
DEBUG = False
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')
SECRET_KEY = env('SECRET_KEY')  # never hardcoded

# Security headers
SECURE_HSTS_SECONDS = 31536000
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
X_FRAME_OPTIONS = 'DENY'

# DRF defaults
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'user': '1000/day',
        'anon': '100/day',
    },
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 15,
    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend'],
}
```

---

## Common Anti-Patterns

```python
# REJECT: No permission class
class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()  # returns ALL orders!

# REJECT: Trusting user input for user field
def perform_create(self, serializer):
    serializer.save()  # user could be set from request body

# REJECT: Raw SQL with string formatting
Order.objects.raw(f"SELECT * FROM orders WHERE id = {order_id}")
# USE: parameterized
Order.objects.raw("SELECT * FROM orders WHERE id = %s", [order_id])

# REJECT: DEBUG=True in production check
if settings.DEBUG:
    pass  # never assume this is False in production
```

---

## Testing Requirements

```python
class OrderAPITest(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.client.force_authenticate(user=self.user)

    def test_list_returns_only_own_orders(self):
        own = OrderFactory(user=self.user)
        other = OrderFactory()  # different user
        response = self.client.get('/api/v1/orders/')
        self.assertEqual(response.status_code, 200)
        ids = [o['id'] for o in response.data['results']]
        self.assertIn(own.id, ids)
        self.assertNotIn(other.id, ids)

    def test_unauthenticated_returns_401(self):
        self.client.force_authenticate(user=None)
        response = self.client.get('/api/v1/orders/')
        self.assertEqual(response.status_code, 401)
```

---

## Deployment Gates

- [ ] `DEBUG=False` in production
- [ ] All secrets from environment (use `django-environ`)
- [ ] `python manage.py check --deploy` passes with no warnings
- [ ] Default permission class is `IsAuthenticated` (not `AllowAny`)
- [ ] All querysets use `select_related`/`prefetch_related` appropriately
- [ ] Database connection pooling configured (PgBouncer or `CONN_MAX_AGE`)
- [ ] Celery configured for async tasks (not sync in request cycle)
- [ ] `collectstatic` run in CI/CD pipeline
