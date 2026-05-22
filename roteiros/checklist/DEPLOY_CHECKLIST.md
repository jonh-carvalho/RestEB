# Deploy Checklist - Django REST em AWS Beanstalk

## ✅ Pré-requisitos Técnicos

### AWS Account & Tools
- [ ] AWS Account ativo
- [ ] IAM User com permissões: `AWSElasticBeanstalkFullAccess`, `IAMFullAccess`, `EC2FullAccess`, `RDSFullAccess`
- [ ] AWS CLI instalado e configurado (`aws configure`)
- [ ] EB CLI instalado (`pip install awsebcli`)
- [ ] Credenciais AWS testadas (`aws s3 ls`)

### Git & Repository
- [ ] Projeto em Git repository
- [ ] `.gitignore` atualizado (exclui `.env`, `*.log`, `db.sqlite3`)
- [ ] Branch `main`/`master` limpo (sem uncommitted changes)
- [ ] Todos os arquivos críticos commitados

## ✅ Arquivos Necessários

### Raiz do Projeto
- [ ] `Procfile` ✅ criado com Gunicorn (4 workers, timeout 60)
- [ ] `runtime.txt` ✅ criado com Python 3.12
- [ ] `requirements.txt` ✅ atualizado com todas as dependências
- [ ] `manage.py` ✅ presente

### Diretório `.ebextensions/`
- [ ] `.ebextensions/01_django.config` ✅ criado (migrations, collectstatic)
- [ ] `.ebextensions/02_static.config` ✅ criado (configuração de estáticos)
- [ ] `.ebextensions/03_environment.config` ✅ criado (health reporting)

### Application Código
- [ ] `catalogo/wsgi.py` ✅ configurado
- [ ] `catalogo/settings.py` ✅ com ambiente support
- [ ] `catalogo/urls.py` ✅ com healthcheck
- [ ] `produtos/` ✅ app configurada

## ✅ Django Configuration

### Security
- [ ] `SECRET_KEY` definida via variável de ambiente
- [ ] `DEBUG = False` em produção (via `DJANGO_DEBUG` env var)
- [ ] `ALLOWED_HOSTS` configurado (via `DJANGO_ALLOWED_HOSTS` env var)
- [ ] `SECURE_SSL_REDIRECT = True` em produção
- [ ] `SESSION_COOKIE_SECURE = True`
- [ ] `CSRF_COOKIE_SECURE = True`
- [ ] HSTS habilitado (31536000 segundos)
- [ ] Proxy SSL header configurado (`X-Forwarded-Proto`)

### Database
- [ ] RDS MySQL preparado ou CloudFormation template pronto
- [ ] Variáveis `RDS_*` documentadas
- [ ] Conexão testada localmente com RDS credentials
- [ ] Migrations criadas: `python manage.py makemigrations`
- [ ] Migrations testadas localmente: `python manage.py migrate`

### Static Files
- [ ] `STATIC_ROOT` configurado em `settings.py`
- [ ] Arquivos estáticos coletados: `python manage.py collectstatic --noinput`
- [ ] Diretório `staticfiles/` populado
- [ ] WhiteNoise considerado (opcional, mas recomendado)

### Media Files
- [ ] Diretório `media/` configurado em `settings.py`
- [ ] S3 setup documentado (se usar)

## ✅ Environment Variables

### Obrigatórias
- [ ] `SECRET_KEY` - Gerada e única
- [ ] `DJANGO_DEBUG=False` - Desabilitado em produção
- [ ] `DJANGO_ALLOWED_HOSTS` - Domínios da aplicação
- [ ] `RDS_DB_NAME` - Nome do banco
- [ ] `RDS_USERNAME` - Usuário RDS
- [ ] `RDS_PASSWORD` - Senha RDS (forte, 16+ chars)
- [ ] `RDS_HOSTNAME` - Endpoint RDS
- [ ] `RDS_PORT` - Porta RDS (padrão 3306)

### Recomendadas
- [ ] `DJANGO_SECURE_SSL_REDIRECT=True`
- [ ] `DJANGO_SECURE_HSTS_SECONDS=31536000`
- [ ] `DJANGO_CSRF_TRUSTED_ORIGINS` - Origens CSRF
- [ ] `DJANGO_SECURE_HSTS_INCLUDE_SUBDOMAINS=True`
- [ ] `DJANGO_SECURE_HSTS_PRELOAD=True`

### Opcionais
- [ ] `USE_S3=False` - Se usar S3
- [ ] `AWS_STORAGE_BUCKET_NAME` - Bucket S3
- [ ] `DJANGO_CORS_ALLOWED_ORIGINS` - Se usar CORS

## ✅ AWS Infrastructure

### RDS MySQL
- [ ] Instância RDS criada (t3.micro para dev, t3.small+ para prod)
- [ ] Engine: MySQL 8.0 ou 5.7
- [ ] Security group RDS permite porta 3306
- [ ] Backup habilitado
- [ ] Multi-AZ considerado (para HA)
- [ ] Parameter Group: `STRICT_TRANS_TABLES` habilitado

### VPC & Security
- [ ] VPC selecionada para RDS e Beanstalk (mesma VPC)
- [ ] Security Group RDS permite Beanstalk security group
- [ ] Security Group Beanstalk permite entrada na porta 80/443
- [ ] IAM Role para Beanstalk tem permissões RDS

### Load Balancer
- [ ] Type: Application Load Balancer
- [ ] HTTPS listener na porta 443 (após certificado)
- [ ] HTTP listener na porta 80 (redirecionar para HTTPS)
- [ ] Health check path: `/` ou `/health/`
- [ ] Health check interval: 30s

## ✅ Testing Pré-Deploy

### Local Testing
- [ ] Projeto roda localmente: `python manage.py runserver`
- [ ] Migrations funcionam: `python manage.py migrate`
- [ ] Collectstatic funciona: `python manage.py collectstatic`
- [ ] Admin acessível: http://localhost:8000/admin
- [ ] API acessível: http://localhost:8000/api/
- [ ] Healthcheck funciona: http://localhost:8000/

### Gunicorn Local Testing
- [ ] Gunicorn roda: `gunicorn catalogo.wsgi:application --workers 4`
- [ ] Porta 8000 acessível: http://localhost:8000/
- [ ] Static files servidos corretamente

### Database Testing
- [ ] Conexão RDS testada: `mysql -h HOSTNAME -u USERNAME -p`
- [ ] Database criado no RDS
- [ ] Django consegue conectar RDS (localmente com SSH tunnel)

## ✅ Documentation

- [ ] README.md com instruções de setup local
- [ ] `AWS_BEANSTALK_COMPLIANCE_REPORT.md` revisado
- [ ] `DEPLOY_GUIDE.md` disponível
- [ ] `IMPROVEMENTS.md` com recomendações
- [ ] Credenciais NOT documentadas (use `.env`)

## ✅ Git & Version Control

- [ ] Branch `deploy-prep` criada (opcional)
- [ ] Todos os commits feitos
- [ ] `.gitignore` atualizado:
  ```
  .env
  *.log
  db.sqlite3
  media/
  staticfiles/
  .venv/
  .mypy_cache/
  .pytest_cache/
  ```
- [ ] Nenhum arquivo sensível commitado
- [ ] Tag de release criada (ex: `v1.0.0`)

## ✅ DNS & Domain (se usar domínio custom)

- [ ] Domínio registrado (Route53 ou registradora externa)
- [ ] Certificate SSL solicitado em ACM
- [ ] Certificate aprovado (validação DNS ou email)
- [ ] Route53 records criados (ou CNAME apontando para Beanstalk)

## ✅ Monitoring & Logging

- [ ] CloudWatch logs habilitados em Beanstalk
- [ ] Log retention: 7-30 dias (definir em `.ebextensions`)
- [ ] Application insights habilitado
- [ ] Email alerts configurados para erros críticos

## ✅ Backup & Disaster Recovery

- [ ] RDS backup habilitado (diário)
- [ ] Retenção de backup: mínimo 7 dias
- [ ] Teste de restore (uma vez)
- [ ] Plano de DR documentado

## 🚀 Deployment Day

### 1. Final Checks (30 min antes)
- [ ] Todas as boxes acima marcadas ✅
- [ ] `.env.prod` preparado (local, não commitado)
- [ ] RDS está up e running
- [ ] Certificado SSL aprovado (se usar custom domain)
- [ ] Time notificado

### 2. Initial Deployment
```bash
# Navegar para projeto
cd /path/to/RestEB

# Inicializar EB (primeira vez)
eb init -p python-3.12 --region us-east-1 --application catalogo-api

# Criar ambiente
eb create production --instance-type t3.small --envvars-file .env.prod

# Monitorar
eb events -f
```

### 3. Post-Deployment Verification (20 min)
- [ ] `eb status` mostra "Ready"
- [ ] `eb open` abre aplicação
- [ ] Health check retorna `{"status": "ok"}`
- [ ] API endpoints respondendo
- [ ] Admin panel acessível
- [ ] Logs sem erros críticos: `eb logs`

### 4. DNS/Domain Setup (se custom domain)
- [ ] Route53 record criado apontando para ALB
- [ ] DNS propagado (verificar com `nslookup` ou `dig`)
- [ ] HTTPS funcionando no domínio

### 5. Monitoring Setup (se não automático)
- [ ] CloudWatch dashboard criado
- [ ] Alarms configurados:
  - [ ] CPU > 70%
  - [ ] Memory > 80%
  - [ ] HTTP 5xx > 10/min
  - [ ] Database connection errors
- [ ] SNS topic para notificações

## 📋 Post-Deployment (Próximas 24h)

### Immediate (1h)
- [ ] Teste de load mínimo (ou synthetics)
- [ ] Verificar logs para warnings/errors
- [ ] Confirmar RDS recebendo conexões
- [ ] Testar endpoints críticos

### Short-term (24h)
- [ ] Monitorar métricas CloudWatch
- [ ] Verificar email/alerts funcionando
- [ ] Testar rollback procedure (não execute!)
- [ ] Documentar lições aprendidas

### Follow-up (1 semana)
- [ ] Revisar performance (latência, CPU, memória)
- [ ] Otimizar se necessário
- [ ] Considerar scaling rules
- [ ] Backup restore test

## 📞 Emergency Procedures

### If Deploy Fails
```bash
eb abort                    # Parar deploy em andamento
eb events -f               # Ver eventos
eb logs                    # Coletar logs
eb ssh                     # SSH na instância para debug
tail -f /var/log/eb-engine.log
```

### If Application Crashes
```bash
eb health                  # Ver status de saúde
eb config                  # Editar configuração
eb terminate production    # Última opção: destruir ambiente
```

### Rollback
```bash
eb appversion              # Listar versões
eb deploy --version <old_version>  # Fazer rollback
```

## 📊 Performance Targets (SLA)

- [ ] Response time: < 200ms (p95)
- [ ] Error rate: < 0.1%
- [ ] Availability: > 99.9%
- [ ] CPU usage: < 50% average
- [ ] Memory usage: < 60% average

---

## ✨ Success Checklist

- [ ] Aplicação deployed e rodando
- [ ] Healthcheck verde
- [ ] API respondendo
- [ ] Banco de dados conectado
- [ ] Logs monitorados
- [ ] Alertas configurados
- [ ] Backup verificado
- [ ] Team notificado
- [ ] Celebrar! 🎉

---

**Data de Deploy:** _______________  
**Deployado por:** _______________  
**Versão:** _______________  
**Problemas encontrados:** _______________  

---

*Para mais detalhes, ver DEPLOY_GUIDE.md*
