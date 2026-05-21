Created 3 todos

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/requirements.txt)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/Procfile)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/runtime.txt)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/catalogo/settings.py#1-1), lines 1 to 120

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/catalogo/settings.py#121-121), lines 121 to 250

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/deploy)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/catalogo/urls.py)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/catalogo/settings.py#60-60), lines 60 to 250

Searched for files matching `**/.ebextensions/**`, 5 matches

Searched for files matching `**/.env`, no matches

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/catalogo/wsgi.py)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/.ebextensions/02_static.config)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/.ebextensions/01_django.config)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/.ebextensions/03_environment.config)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/.ebextensions/django.config)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/.ebextensions/detection.config)

Searched for text `STATIC_ROOT|STATIC_URL|STATICFILES|whitenoise|django-storages` (`**/*.py`), 5 results

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/produtos/views.py)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/produtos/models.py)

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/roteiros/checklist-console-aws-appzip.md#1-1), lines 1 to 40

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/roteiros/checklist/DEPLOY_CHECKLIST.md#1-1), lines 1 to 60

Searched for text `WSGIPath` (`**/*.config`), 2 results

Searched for files matching `**/.gitignore`, 1 match

Searched for text `whitenoise|django-storages|boto3` (`**/requirements.txt`), no results

Ran terminal command: if (Test-Path "deploy\app.zip") { Add-Type -AssemblyName System.IO.Compression.FileSystem ; $zip = [System.IO.Compression.ZipFile]::OpenRead("deploy\app.zip") ; $zip.Entries | Select-Object FullName | Sort-Object FullName } else { Write-Host "app.zip not found" }

Read [](file:///c%3A/Users/00661711722/Downloads/RestEB/.gitignore#1-1), lines 1 to 40

Ran terminal command: Add-Type -AssemblyName System.IO.Compression.FileSystem ; $zip = [System.IO.Compression.ZipFile]::OpenRead("deploy\app.zip") ; $zip.Entries | Where-Object { $_.FullName -match '\.ebextensions' } | Select-Object FullName

Searched for files matching `**/.platform/**`, no matches

Starting: *Verificar conformidade e gerar relatório* (3/3)

Aqui está o relatório completo de conformidade do workspace para deploy no AWS Elastic Beanstalk via Console:

---

## Relatório de Conformidade — Deploy Django REST no AWS Elastic Beanstalk

### CRÍTICO — Bloqueadores de Deploy

**1. app.zip está incompleto**

O zip contém apenas 2 configs .ebextensions:
- ✅ django.config
- ✅ detection.config

Faltam **3 arquivos** que existem no workspace mas não foram incluídos no zip:
- ❌ 01_django.config — migrate com ativação correta do venv
- ❌ 02_static.config — paths absolutos de static (`/var/app/current/staticfiles`)
- ❌ 03_environment.config — health reporting e rolling update
- ❌ runtime.txt — ausente no zip; sem ele o EB pode usar Python errado

**2. Gunicorn bind na porta errada — causará 502 Bad Gateway**

```
Procfile: --bind 0.0.0.0:8000
```
O nginx do EB (AL2/AL2023) faz proxy para a porta **8080** por padrão. Bindar em 8000 causará erro 502 em todas as requisições.

**Correção:**
```
web: gunicorn catalogo.wsgi:application --workers 4 --bind 0.0.0.0:8080 --timeout 60 --access-logfile - --error-logfile -
```

**3. `mysqlclient` requer bibliotecas nativas não presentes no EB**

`mysqlclient==2.2.8` depende de `libmysqlclient-dev` / `mysql-devel` no SO. O EB não os instala por padrão. O `pip install` vai falhar no deploy.

**Correção** — adicionar ao .ebextensions:
```yaml
packages:
  yum:
    mysql-devel: []
    python3-devel: []
    gcc: []
```

---

### ALTO — Podem causar falha ou comportamento incorreto

**4. Conflito de `WSGIPath` entre configs**

| Arquivo | WSGIPath |
|---|---|
| django.config | wsgi.py |
| 01_django.config | `catalogo/wsgi:application` |

O formato correto para plataformas modernas (AL2/AL2023) é `catalogo.wsgi:application`. Mas como o Procfile existe, o EB ignora o WSGIPath — desde que a porta seja a correta (item 2 acima).

**5. `container_commands` sem ativação do venv**

No django.config (único no zip):
```yaml
container_commands:
  01_migrate:
    command: "python manage.py migrate --noinput"  # python sem venv ativo
```
Em AL2/AL2023 o `python` do sistema não é o venv. O comando correto é:
```yaml
command: "source /var/app/venv/*/bin/activate && python manage.py migrate --noinput"
```

**6. config.yml incluído no zip**

Este arquivo é para o EB CLI e não deve estar no zip de upload manual. Pode causar conflitos de configuração.

**7. `__pycache__` no zip com bytecode de Python 3.13/3.14**

O zip contém `.pyc` compilados para Python 3.13 e 3.14, mas o runtime.txt define `python-3.12.1`. O EB vai regenerar os `.pyc` mas a presença de bytecode incompatível pode gerar warnings ou conflitos.

**8. detection.config usa variáveis não-padrão do EB**

```yaml
AWS_EB_PLATFORM_DETECTION: "python"
AWS_EB_PLATFORM_OVERRIDE: "Python 3.12"
```
Essas chaves **não existem** na API do Elastic Beanstalk. São ignoradas silenciosamente mas não fazem o efeito esperado.

---

### MÉDIO — Riscos operacionais

**9. Media files sem persistência**

O modelo `Produto` usa `ImageField` mas os arquivos de mídia ficam em disco local da instância EC2. Ao trocar instância (scale/deploy), os arquivos são perdidos. Recomendado: configurar S3 com `django-storages`.

**10. `DJANGO_ALLOWED_HOSTS` deve ser definido manualmente no console**

A variável é obrigatória no settings.py e não está em nenhum .ebextensions. Deve ser configurada via "Environment Properties" no console EB antes do primeiro deploy.

**11. `CSRF_TRUSTED_ORIGINS` não configurado**

Sem `CSRF_TRUSTED_ORIGINS`, requisições POST pela URL do EB (`*.elasticbeanstalk.com`) serão bloqueadas por CSRF. Deve incluir o domínio EB no console.

**12. `SECURE_SSL_REDIRECT=True` sem HTTPS no EB**

Se o environment EB não tiver listener HTTPS (porta 443) configurado no Load Balancer, o redirect vai gerar loop infinito. Desativar até configurar certificado: `DJANGO_SECURE_SSL_REDIRECT=False`.

---

### OK — Itens conformes ✅

| Item | Status |
|---|---|
| manage.py no zip | ✅ |
| requirements.txt no zip | ✅ |
| Procfile no zip | ✅ |
| catalogo e produtos no zip | ✅ |
| wsgi.py configurado corretamente | ✅ |
| `SECRET_KEY` via variável de ambiente | ✅ |
| `DEBUG=False` configurável por env | ✅ |
| `ALLOWED_HOSTS` via env | ✅ |
| `STATIC_ROOT` definido (`staticfiles/`) | ✅ |
| `collectstatic` no deploy | ✅ |
| `migrate` no deploy | ✅ |
| Suporte a RDS (variáveis `RDS_*`) | ✅ |
| Healthcheck em `/` | ✅ |
| `SECURE_PROXY_SSL_HEADER` configurado | ✅ |
| Migrações presentes (`0001_initial.py`) | ✅ |

---

### Ações Prioritárias antes do Deploy

1. **Corrigir Procfile** — mudar porta para `8080`
2. **Adicionar yum packages** no .ebextensions para `mysqlclient`
3. **Recriar app.zip** incluindo: runtime.txt, 01_django.config, 02_static.config, 03_environment.config, e excluindo .elasticbeanstalk e `__pycache__`
4. **Corrigir `container_commands`** no django.config para ativar o venv
5. **Definir no console EB**: `DJANGO_ALLOWED_HOSTS`, `SECRET_KEY`, `RDS_*`, `DJANGO_SECURE_SSL_REDIRECT=False` (temporariamente)

