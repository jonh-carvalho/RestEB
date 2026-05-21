# Melhorias Recomendadas para Production-Ready

## 1. Healthcheck Robusto

Atualizar `catalogo/urls.py`:

```python
from django.contrib import admin
from django.http import JsonResponse
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static
from django.db import connection

def healthcheck(request):
    """
    Healthcheck que valida:
    - Status da aplicação
    - Conexão com banco de dados
    - Migrations aplicadas
    """
    try:
        # Validar conexão com banco de dados
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        
        return JsonResponse({
            'status': 'ok',
            'database': 'connected',
            'debug': settings.DEBUG
        })
    except Exception as e:
        return JsonResponse({
            'status': 'error',
            'database': 'disconnected',
            'error': str(e)
        }, status=503)

urlpatterns = [
    path('', healthcheck),
    path('health/', healthcheck),  # AWS ALB health check
    path('admin/', admin.site.urls),
    path('api/', include('produtos.urls')),
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

## 2. Logging Estruturado

Adicionar ao final de `catalogo/settings.py`:

```python
# Logging Configuration for Production
import logging

LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
        'json': {
            '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(levelname)s %(asctime)s %(name)s %(message)s'
        }
    },
    'filters': {
        'require_debug_false': {
            '()': 'django.utils.log.RequireDebugFalse',
        },
        'require_debug_true': {
            '()': 'django.utils.log.RequireDebugTrue',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
        'file': {
            'level': 'INFO',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': os.path.join(BASE_DIR, 'logs', 'app.log'),
            'maxBytes': 1024 * 1024 * 15,  # 15MB
            'backupCount': 10,
            'formatter': 'verbose',
        },
        'django_file': {
            'level': 'INFO',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': os.path.join(BASE_DIR, 'logs', 'django.log'),
            'maxBytes': 1024 * 1024 * 15,  # 15MB
            'backupCount': 10,
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'django_file'],
            'level': 'INFO',
            'propagate': False,
        },
        'django.request': {
            'handlers': ['console', 'django_file'],
            'level': 'WARNING',
            'propagate': False,
        },
        'catalogo': {
            'handlers': ['console', 'file'],
            'level': 'DEBUG' if DEBUG else 'INFO',
            'propagate': False,
        },
        'produtos': {
            'handlers': ['console', 'file'],
            'level': 'DEBUG' if DEBUG else 'INFO',
            'propagate': False,
        },
    },
    'root': {
        'handlers': ['console', 'file'],
        'level': 'INFO',
    },
}

# Criar diretório de logs se não existir
import os
logs_dir = os.path.join(BASE_DIR, 'logs')
os.makedirs(logs_dir, exist_ok=True)
```

## 3. CORS Configuração (se necessário)

```python
# Em settings.py
INSTALLED_APPS = [
    # ... outras apps ...
    'corsheaders',  # Adicionar antes de outros
    # ... resto ...
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',  # Adicionar no topo
    'django.middleware.security.SecurityMiddleware',
    # ... resto ...
]

# CORS Settings
CORS_ALLOWED_ORIGINS = [
    origin.strip()
    for origin in os.getenv('DJANGO_CORS_ALLOWED_ORIGINS', '').split(',')
    if origin.strip()
]

CORS_ALLOW_CREDENTIALS = True
```

## 4. WhiteNoise para Estáticos

```python
# Em settings.py MIDDLEWARE
MIDDLEWARE = [
    'whitenoise.middleware.WhiteNoiseMiddleware',  # Adicionar logo após SecurityMiddleware
    'django.middleware.security.SecurityMiddleware',
    # ... resto ...
]

# Compressão de assets estáticos
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

## 5. S3 Storage para Media (Opcional)

```python
# Em settings.py, após configuração de MEDIA
if not DEBUG and os.getenv('USE_S3', 'False') == 'True':
    # AWS S3 Configuration
    AWS_STORAGE_BUCKET_NAME = os.getenv('AWS_STORAGE_BUCKET_NAME')
    AWS_S3_REGION_NAME = os.getenv('AWS_S3_REGION_NAME', 'us-east-1')
    AWS_S3_CUSTOM_DOMAIN = f'{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com'
    
    MEDIA_URL = f'https://{AWS_S3_CUSTOM_DOMAIN}/media/'
    DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
    
    # Otimizações S3
    AWS_S3_OBJECT_PARAMETERS = {
        'CacheControl': 'max-age=86400',
    }
```

## 6. Gzip Compression

```python
# Em settings.py MIDDLEWARE
MIDDLEWARE = [
    'django.middleware.gzip.GZipMiddleware',  # Adicionar no topo após WhiteNoise
    'whitenoise.middleware.WhiteNoiseMiddleware',
    # ... resto ...
]
```

## 7. Variáveis de Ambiente Recomendadas

Adicionar ao `.env` ou Beanstalk environment variables:

```bash
# Core
SECRET_KEY=generated-key
DJANGO_DEBUG=False
DJANGO_SETTINGS_MODULE=catalogo.settings

# Database
RDS_DB_NAME=catalogo_prod
RDS_USERNAME=admin
RDS_PASSWORD=strong-password
RDS_HOSTNAME=catalogo-db.c9akciq32.us-east-1.rds.amazonaws.com
RDS_PORT=3306

# Hosts
DJANGO_ALLOWED_HOSTS=seu-app.elasticbeanstalk.com,seu-dominio.com,www.seu-dominio.com

# Security
DJANGO_SECURE_SSL_REDIRECT=True
DJANGO_SECURE_HSTS_SECONDS=31536000
DJANGO_SECURE_HSTS_INCLUDE_SUBDOMAINS=True
DJANGO_SECURE_HSTS_PRELOAD=True
DJANGO_CSRF_TRUSTED_ORIGINS=https://seu-dominio.com,https://www.seu-dominio.com

# CORS (se necessário)
DJANGO_CORS_ALLOWED_ORIGINS=https://seu-frontend.com

# S3 (opcional)
USE_S3=False
AWS_STORAGE_BUCKET_NAME=seu-bucket
AWS_S3_REGION_NAME=us-east-1

# Email (se implementar)
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.ses.amazonaws.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=seu-email@dominio.com
EMAIL_HOST_PASSWORD=senha-ses
```

## 8. Estrutura de Diretórios para Produção

```
RestEB/
├── logs/                    # Criado automaticamente
│   ├── app.log
│   ├── django.log
│   └── eb-engine.log
├── staticfiles/             # Coletado via collectstatic
├── media/                   # Ou em S3
├── catalogo/
│   ├── settings.py          # Com logging
│   ├── wsgi.py
│   └── urls.py              # Com healthcheck melhorado
├── produtos/
└── .ebextensions/
```

## 9. RDS Configuration (MySQL Best Practices)

Em `.ebextensions/04_mysql.config`:

```yaml
option_settings:
  aws:rds:db:
    DBAllocatedStorage: 20
    DBEngine: mysql
    DBEngineVersion: 8.0.35
    DBInstanceClass: db.t3.micro
    DBUser: admin
    MultiAZ: false
    StorageType: gp3
    
    # Para produção real, considere:
    # - MultiAZ: true
    # - BackupRetentionPeriod: 30
    # - PreferredBackupWindow: "03:00-04:00"
    # - PreferredMaintenanceWindow: "sun:04:00-sun:05:00"
```

## 10. Monitoring com CloudWatch

Em `.ebextensions/05_cloudwatch.config`:

```yaml
option_settings:
  aws:elasticbeanstalk:cloudwatch:logs:
    StreamLogs: true
    DeleteOnTerminate: false
    RetentionInDays: 30
  
  aws:elasticbeanstalk:xray:
    XRayEnabled: false  # Ativar se precisar de distributed tracing

files:
  "/opt/elasticbeanstalk/tasks/bundlelogs.d/custom_metrics.py":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/usr/bin/env python
      # Custom CloudWatch metrics pode ser adicionado aqui
      import boto3
      
      cloudwatch = boto3.client('cloudwatch')
      # Implementar métricas customizadas conforme necessário
```

---

Estas melhorias prepararam melhor a aplicação para produção em AWS Beanstalk com:
- ✅ Healthcheck robusto
- ✅ Logging estruturado
- ✅ Suporte a CORS
- ✅ Estáticos otimizados com WhiteNoise
- ✅ Configuração S3 (opcional)
- ✅ Compressão Gzip
- ✅ Monitoramento CloudWatch

**Próximo passo:** Atualizar `requirements.txt` com dependências recomendadas e fazer deploy!
