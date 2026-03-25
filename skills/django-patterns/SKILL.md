---
name: django-patterns
description: Django architecture patterns, REST API design with DRF, ORM best practices, caching, signals, middleware, and production-grade Django apps.
origin: ECC
---

# Django Development Patterns

Production-grade Django architecture patterns for scalable, maintainable applications.

## When to Activate

- Building Django web applications
- Designing Django REST Framework APIs
- Working with Django ORM and models
- Setting up Django project structure
- Implementing caching, signals, middleware

## Project Structure

### Recommended Layout

```
myproject/
├── config/
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── development.py
│   │   ├── production.py
│   │   └── test.py
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
├── manage.py
└── apps/
    ├── __init__.py
    ├── users/
    │   ├── __init__.py
    │   ├── models.py
    │   ├── views.py
    │   ├── serializers.py
    │   ├── urls.py
    │   ├── permissions.py
    │   ├── filters.py
    │   ├── services.py
    │   └── tests/
    └── products/
        └── ...
```

### Split Settings Pattern

```python
# config/settings/base.py
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

SECRET_KEY = env('DJANGO_SECRET_KEY')
DEBUG = False
ALLOWED_HOSTS = []

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'rest_framework.authtoken',
    'corsheaders',
    # Local apps
    'apps.users',
    'apps.products',
]

# config/settings/development.py
from .base import *

DEBUG = True
ALLOWED_HOSTS = ['localhost', '127.0.0.1']

# config/settings/production.py
from .base import *

DEBUG = False
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
```

## Model Design Patterns

### Custom QuerySet

```python
class ProductQuerySet(models.QuerySet):
    def active(self):
        return self.filter(is_active=True)

    def with_category(self):
        return self.select_related('category')

    def with_tags(self):
        return self.prefetch_related('tags')

    def in_stock(self):
        return self.filter(stock__gt=0)

    def search(self, query):
        return self.filter(
            models.Q(name__icontains=query) |
            models.Q(description__icontains=query)
        )

class Product(models.Model):
    objects = ProductQuerySet.as_manager()

# Usage
Product.objects.active().with_category().in_stock()
```

## Django REST Framework Patterns

### ViewSet with Serializers

```python
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.select_related('category').prefetch_related('tags')
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]

    def get_serializer_class(self):
        if self.action == 'create':
            return ProductCreateSerializer
        return ProductSerializer

    def perform_create(self, serializer):
        serializer.save(created_by=self.request.user)

    @action(detail=False, methods=['get'])
    def featured(self, request):
        featured = self.queryset.filter(is_featured=True)[:10]
        serializer = self.get_serializer(featured, many=True)
        return Response(serializer.data)
```

## Service Layer Pattern

```python
class OrderService:
    @staticmethod
    @transaction.atomic
    def create_order(user, cart: Cart) -> Order:
        order = Order.objects.create(user=user, total_price=cart.total_price)
        for item in cart.items.all():
            OrderItem.objects.create(
                order=order, product=item.product,
                quantity=item.quantity, price=item.product.price
            )
        cart.items.all().delete()
        return order
```

## Performance Optimization

### N+1 Query Prevention

```python
# Bad - N+1 queries
products = Product.objects.all()
for product in products:
    print(product.category.name)

# Good - Single query with select_related
products = Product.objects.select_related('category').all()

# Good - Prefetch for many-to-many
products = Product.objects.prefetch_related('tags').all()
```

## Quick Reference

| Pattern | Description |
|---------|-------------|
| Split settings | Separate dev/prod/test settings |
| Custom QuerySet | Reusable query methods |
| Service Layer | Business logic separation |
| ViewSet | REST API endpoints |
| select_related | Foreign key optimization |
| prefetch_related | Many-to-many optimization |
| Cache first | Cache expensive operations |
| Signals | Event-driven actions |
| Middleware | Request/response processing |
