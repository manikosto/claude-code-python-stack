---
name: django-tdd
description: Django testing strategies with pytest-django, TDD methodology, factory_boy, mocking, coverage, and testing Django REST Framework APIs.
origin: ECC
---

# Django Testing with TDD

Test-driven development for Django applications using pytest, factory_boy, and Django REST Framework.

## When to Activate

- Writing new Django applications
- Implementing Django REST Framework APIs
- Testing Django models, views, and serializers

## Setup

### pytest Configuration

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.test
testpaths = tests
addopts = --reuse-db --nomigrations --cov=apps --cov-report=html --strict-markers
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
```

### conftest.py

```python
import pytest
from django.contrib.auth import get_user_model

User = get_user_model()

@pytest.fixture
def user(db):
    return User.objects.create_user(email='test@example.com', password='testpass123', username='testuser')

@pytest.fixture
def authenticated_client(client, user):
    client.force_login(user)
    return client

@pytest.fixture
def api_client():
    from rest_framework.test import APIClient
    return APIClient()

@pytest.fixture
def authenticated_api_client(api_client, user):
    api_client.force_authenticate(user=user)
    return api_client
```

## Factory Boy

```python
import factory

class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User

    email = factory.Sequence(lambda n: f"user{n}@example.com")
    username = factory.Sequence(lambda n: f"user{n}")
    password = factory.PostGenerationMethodCall('set_password', 'testpass123')

class ProductFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Product

    name = factory.Faker('sentence', nb_words=3)
    price = factory.fuzzy.FuzzyDecimal(10.00, 1000.00, 2)
    category = factory.SubFactory(CategoryFactory)
```

## DRF API Testing

```python
class TestProductAPI:
    def test_list_products(self, api_client, db):
        ProductFactory.create_batch(10)
        response = api_client.get(reverse('api:product-list'))
        assert response.status_code == status.HTTP_200_OK
        assert response.data['count'] == 10

    def test_create_product_unauthorized(self, api_client, db):
        response = api_client.post(reverse('api:product-list'), {'name': 'Test'})
        assert response.status_code == status.HTTP_401_UNAUTHORIZED

    def test_create_product_authorized(self, authenticated_api_client, db):
        data = {'name': 'Test Product', 'price': '99.99', 'stock': 10}
        response = authenticated_api_client.post(reverse('api:product-list'), data)
        assert response.status_code == status.HTTP_201_CREATED
```

## Coverage Goals

| Component | Target |
|-----------|--------|
| Models | 90%+ |
| Serializers | 85%+ |
| Views | 80%+ |
| Services | 90%+ |
| Overall | 80%+ |
