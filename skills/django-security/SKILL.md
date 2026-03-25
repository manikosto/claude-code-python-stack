---
name: django-security
description: Django security best practices, authentication, authorization, CSRF protection, SQL injection prevention, XSS prevention, and secure deployment configurations.
origin: ECC
---

# Django Security Best Practices

Comprehensive security guidelines for Django applications.

## When to Activate

- Setting up Django authentication and authorization
- Implementing user permissions and roles
- Configuring production security settings
- Reviewing Django application for security issues

## Core Security Settings

```python
# settings/production.py
DEBUG = False
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'

SESSION_COOKIE_HTTPONLY = True
CSRF_COOKIE_HTTPONLY = True

SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY')
if not SECRET_KEY:
    raise ImproperlyConfigured('DJANGO_SECRET_KEY required')

PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.Argon2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',
]
```

## SQL Injection Prevention

```python
# GOOD: Django ORM automatically escapes
User.objects.filter(email__iexact=email)

# GOOD: Parameterized raw queries
User.objects.raw('SELECT * FROM users WHERE username = %s', [query])

# BAD: Never interpolate user input
User.objects.raw(f'SELECT * FROM users WHERE username = {username}')
```

## Custom Permissions (DRF)

```python
class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.author == request.user
```

## API Rate Limiting

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
    }
}
```

## Quick Security Checklist

| Check | Description |
|-------|-------------|
| `DEBUG = False` | Never in production |
| HTTPS only | Force SSL, secure cookies |
| Strong secrets | Environment variables for SECRET_KEY |
| CSRF protection | Enabled by default, don't disable |
| XSS prevention | Auto-escapes, don't use `|safe` with user input |
| SQL injection | Use ORM, never concatenate strings |
| Rate limiting | Throttle API endpoints |
| Security headers | CSP, X-Frame-Options, HSTS |
