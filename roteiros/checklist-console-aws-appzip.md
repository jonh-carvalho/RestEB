# Checklist Exato - Deploy Manual no AWS Elastic Beanstalk via Console

Data de referencia: 2026-05-21

Escopo deste checklist:

- Considera apenas o envio manual do arquivo `deploy/app.zip` pelo Console AWS.
- Nao considera uso de `eb deploy`, `eb init` ou qualquer fluxo via EB CLI.
- Considera o estado atual deste projeto Django.

## 1) Artefato que sera enviado

- [ ] Arquivo selecionado para upload: `deploy/app.zip`
- [ ] Nao enviar a pasta do projeto inteira.
- [ ] Nao enviar `appsql.zip`.
- [ ] O zip enviado contem na raiz:
  - [ ] `manage.py`
  - [ ] `requirements.txt`
  - [ ] `Procfile`
  - [ ] pasta `.ebextensions/`
  - [ ] pasta `catalogo/`
  - [ ] pasta `produtos/`

## 2) Pre-condicoes obrigatorias do projeto

Este projeto so sobe em producao se estas variaveis existirem no ambiente:

- [ ] `SECRET_KEY`
- [ ] `DJANGO_ALLOWED_HOSTS`
- [ ] `RDS_DB_NAME`
- [ ] `RDS_USERNAME`
- [ ] `RDS_PASSWORD`
- [ ] `RDS_HOSTNAME`
- [ ] `RDS_PORT` (recomendado; se omitir, o projeto usa `3306`)

Variaveis importantes recomendadas:

- [ ] `DJANGO_DEBUG=False`
- [ ] `DJANGO_CSRF_TRUSTED_ORIGINS=https://SEU_DOMINIO`
- [ ] `DJANGO_SECURE_SSL_REDIRECT=True`
- [ ] `DJANGO_SECURE_HSTS_SECONDS=31536000`

## 3) Valores exatos para configurar no Console AWS

Em `Configuration > Updates, monitoring, and logging > Edit` ou na tela equivalente de environment properties:

- [ ] `SECRET_KEY=<gere_uma_chave_forte_do_django>`
- [ ] `DJANGO_DEBUG=False`
- [ ] `DJANGO_ALLOWED_HOSTS=<seu-dominio-eb>,.elasticbeanstalk.com`
- [ ] `DJANGO_CSRF_TRUSTED_ORIGINS=https://<seu-dominio-eb>`
- [ ] `DJANGO_SECURE_SSL_REDIRECT=True`
- [ ] `DJANGO_SECURE_HSTS_SECONDS=31536000`
- [ ] `RDS_DB_NAME=<nome_do_banco>`
- [ ] `RDS_USERNAME=<usuario_do_banco>`
- [ ] `RDS_PASSWORD=<senha_do_banco>`
- [ ] `RDS_HOSTNAME=<endpoint_do_rds>`
- [ ] `RDS_PORT=3306`

Observacoes:

- [ ] Substituir `<seu-dominio-eb>` pelo dominio real do ambiente, por exemplo `meu-ambiente.us-east-1.elasticbeanstalk.com`.
- [ ] Se houver dominio proprio, incluir tambem em `DJANGO_ALLOWED_HOSTS`.
- [ ] Se houver dominio proprio com HTTPS, incluir tambem em `DJANGO_CSRF_TRUSTED_ORIGINS`.

## 4) Passo a passo do deploy no Console AWS

### 4.1 Criacao ou selecao do ambiente

- [ ] Abrir AWS Console.
- [ ] Ir em `Elastic Beanstalk`.
- [ ] Criar uma `Application` nova ou abrir a aplicacao existente.
- [ ] Criar ou selecionar um `Environment` do tipo `Web server environment`.
- [ ] Plataforma escolhida: `Python 3.12`.

### 4.2 Upload do codigo

- [ ] Em `Application versions` ou na tela de criacao do ambiente, escolher `Upload your code`.
- [ ] Selecionar `Local file`.
- [ ] Enviar `deploy/app.zip`.
- [ ] Confirmar que o upload terminou sem erro.

### 4.3 Configuracao de software

- [ ] Abrir a configuracao de software do ambiente.
- [ ] Preencher todas as variaveis listadas na secao 3.
- [ ] Salvar a configuracao.

### 4.4 Banco de dados e rede

- [ ] O banco RDS existe e esta ativo.
- [ ] O `RDS_HOSTNAME` aponta para o endpoint correto.
- [ ] O Security Group do RDS permite conexao na porta `3306` a partir das instancias do Elastic Beanstalk.
- [ ] O Security Group do ambiente permite entrada HTTP `80`.
- [ ] Se houver HTTPS no load balancer, permitir entrada HTTPS `443`.

## 5) O que o bundle ja faz automaticamente

O `app.zip` ja leva configuracoes que o Elastic Beanstalk deve aplicar:

- [ ] `Procfile` para iniciar `gunicorn`
- [ ] `.ebextensions/django.config` com:
  - [ ] `DJANGO_SETTINGS_MODULE=catalogo.settings`
  - [ ] `DJANGO_DEBUG=False`
  - [ ] `python manage.py migrate --noinput`
  - [ ] `python manage.py collectstatic --noinput`
  - [ ] mapeamento de `/static`
  - [ ] mapeamento de `/media`

Atencao:

- [ ] O deploy falha se `migrate` rodar sem acesso ao banco.
- [ ] O deploy falha se `SECRET_KEY` nao existir.
- [ ] O deploy falha se `DJANGO_ALLOWED_HOSTS` nao existir.

## 6) Validacao imediata apos publicar

### 6.1 Estado do ambiente

- [ ] O deploy terminou sem rollback.
- [ ] O ambiente ficou com `Health: Green`.
- [ ] Os eventos do ambiente nao mostram erro em `migrate`.
- [ ] Os eventos do ambiente nao mostram erro em `collectstatic`.
- [ ] O processo `web` iniciou com sucesso.

### 6.2 Testes HTTP minimos

- [ ] A URL raiz `/` responde `200`
- [ ] A raiz retorna `{"status": "ok"}`
- [ ] A rota `/api/produtos/` responde sem erro `5xx`
- [ ] O admin `/admin/` abre pelo menos a tela de login

## 7) Itens de aprovacao

### 7.1 Aprovado para aula/demo

- [ ] Upload do `deploy/app.zip` feito pelo Console AWS
- [ ] Variaveis de ambiente obrigatorias preenchidas
- [ ] Banco acessivel
- [ ] Deploy concluido
- [ ] Health `Green`
- [ ] Raiz `/` respondendo `200`
- [ ] API `/api/produtos/` respondendo

### 7.2 Nao aprovado para producao plena sem ajuste adicional

- [ ] Upload de imagens ainda usa `media/` local quando `USE_S3=False`
- [ ] Em Elastic Beanstalk, esse armazenamento nao deve ser tratado como persistente
- [ ] Para producao real, migrar media para S3 antes da aprovacao final

## 8) Bloqueadores objetivos

Se qualquer item abaixo acontecer, nao prosseguir como aprovado:

- [ ] Faltou `SECRET_KEY`
- [ ] Faltou `DJANGO_ALLOWED_HOSTS`
- [ ] Faltou qualquer variavel `RDS_*` obrigatoria
- [ ] O RDS nao aceita conexao a partir do ambiente
- [ ] O ambiente entrou em `Severe` ou `Degraded`
- [ ] A raiz `/` nao respondeu `200`
- [ ] Houve erro de importacao do app ou erro no `gunicorn`

## 9) Evidencias para registrar

- [ ] Print do upload da versao com `app.zip`
- [ ] Print das environment properties preenchidas
- [ ] Print do `Health: Green`
- [ ] Print dos eventos com deploy concluido
- [ ] Print da raiz `/`
- [ ] Print ou teste da rota `/api/produtos/`

## 10) Registro rapido da execucao

- Responsavel:
- Ambiente:
- Regiao AWS:
- Dominio EB:
- Banco RDS:
- Data/hora:
- Resultado final: ( ) Aprovado para aula/demo  ( ) Reprovado  ( ) Pendente ajuste de S3
