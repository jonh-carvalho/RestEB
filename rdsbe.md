# Roteiro Simples: Migrar SQLite para MySQL (RDS) no Elastic Beanstalk

## Objetivo
Migrar a aplicacao Django REST do banco SQLite para MySQL no Amazon RDS, mantendo deploy no Elastic Beanstalk (EB).

## O que foi identificado no projeto atual
- O projeto esta preparado para EB com .ebextensions e Procfile.
- O arquivo de configuracao do banco em catalogo/settings.py precisa ser normalizado antes da migracao.
- requirements.txt ainda nao possui driver MySQL.
- .ebextensions/django.config ainda tem comandos de permissao para SQLite, que deixam de fazer sentido no RDS.

## Parte 1: Preparacao (15 min)

### 1. Criar backup do SQLite
No seu ambiente local:

```bash
copy db.sqlite3 db.sqlite3.bkp
python manage.py dumpdata --natural-foreign --natural-primary -e contenttypes -e auth.Permission > dados_iniciais.json
```

### 2. Criar banco no RDS (MySQL)
No console da AWS:
1. RDS > Create database > MySQL 8.0.
2. Opcao Free tier ou Dev/Test (conforme aula).
3. Anote:
   - endpoint
   - porta (3306)
   - nome do banco
   - usuario
   - senha
4. Security Group do RDS deve aceitar conexao da SG do Elastic Beanstalk na porta 3306.

## Parte 2: Ajustes no codigo (20 min)

### 1. Adicionar driver MySQL
No requirements.txt, adicione:

```txt
PyMySQL==1.1.1
```

### 2. Registrar o PyMySQL
No arquivo catalogo/__init__.py:

```python
import pymysql

pymysql.install_as_MySQLdb()
```

### 3. Padronizar DATABASES no settings
Substitua a configuracao de banco por um bloco unico e simples:

```python
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent
DEBUG = os.getenv('DJANGO_DEBUG', 'True') == 'True'

USE_SQLITE = os.getenv('USE_SQLITE', 'True') == 'True'

if USE_SQLITE:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.sqlite3',
            'NAME': BASE_DIR / 'db.sqlite3',
        }
    }
else:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.mysql',
            'NAME': os.getenv('RDS_DB_NAME', ''),
            'USER': os.getenv('RDS_USERNAME', ''),
            'PASSWORD': os.getenv('RDS_PASSWORD', ''),
            'HOST': os.getenv('RDS_HOSTNAME', ''),
            'PORT': os.getenv('RDS_PORT', '3306'),
            'OPTIONS': {
                'charset': 'utf8mb4',
            },
        }
    }
```

### 4. Limpar comandos legados de SQLite no EB
No arquivo .ebextensions/django.config, remova os comandos:
- 04_permissao_arquivo_sqlite
- 05_permissao_pasta_sqlite

Mantenha migrate e collectstatic.

## Parte 3: Variaveis de ambiente no Elastic Beanstalk (10 min)

Em Configuration > Software > Environment properties:

- DJANGO_DEBUG=False
- USE_SQLITE=False
- DJANGO_ALLOWED_HOSTS=<dominio-do-ambiente>,.elasticbeanstalk.com
- RDS_DB_NAME=<nome_do_banco>
- RDS_USERNAME=<usuario>
- RDS_PASSWORD=<senha>
- RDS_HOSTNAME=<endpoint_rds>
- RDS_PORT=3306

## Parte 4: Migracao de estrutura e dados (20 min)

### 1. Criar estrutura no MySQL
No ambiente local, testando com as mesmas variaveis do RDS:

```bash
python manage.py migrate
```

### 2. Carga de dados no MySQL

```bash
python manage.py loaddata dados_iniciais.json
```

### 3. Validacao local

```bash
python manage.py runserver
```

Validar:
- / (healthcheck)
- /api/produtos/
- /admin/

## Parte 5: Deploy no Elastic Beanstalk (10 min)

1. Gerar zip sem venv, sem db.sqlite3 e sem __pycache__.
2. Upload no EB (Python 3.12).
3. Aguardar deploy.
4. Conferir logs e status Green.

## Parte 6: Checklist rapido

1. requirements.txt com PyMySQL.
2. catalogo/__init__.py com install_as_MySQLdb().
3. settings.py sem conflito e com bloco unico de DATABASES.
4. USE_SQLITE=False no EB.
5. Variaveis RDS_* preenchidas.
6. RDS liberado para SG do EB na porta 3306.
7. Deploy Green e endpoints respondendo.

## Rollback simples
Se algo falhar:
1. Restaurar versao anterior da aplicacao no EB.
2. Reativar temporariamente USE_SQLITE=True apenas localmente para diagnostico.
3. Ajustar variaveis/seguranca do RDS e redeploy.

---

Se quiser, no proximo passo eu aplico automaticamente essas alteracoes nos arquivos do projeto (settings.py, requirements.txt, catalogo/__init__.py e .ebextensions/django.config) ja no formato final para deploy.