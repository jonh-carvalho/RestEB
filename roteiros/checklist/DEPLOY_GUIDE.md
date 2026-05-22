# Guia de Deploy - Django REST em AWS Beanstalk

## 📋 Pré-requisitos

```bash
# 1. Instalar AWS CLI
pip install awscli

# 2. Instalar EB CLI
pip install awsebcli

# 3. Verificar credenciais AWS
aws configure
```

## 🚀 Passo a Passo do Deploy

### 1. Preparar Variáveis de Ambiente

Crie um arquivo `.env.prod` com as seguintes variáveis (não comite este arquivo):

```bash
# Segurança
SECRET_KEY=sua-chave-secreta-gerada
DJANGO_DEBUG=False
DJANGO_SECURE_SSL_REDIRECT=True

# Hosts
DJANGO_ALLOWED_HOSTS=seu-app.elasticbeanstalk.com,seu-dominio.com

# Banco de Dados RDS
RDS_DB_NAME=catalogo_db
RDS_USERNAME=admin
RDS_PASSWORD=sua-senha-forte
RDS_HOSTNAME=seu-db-instance.c9akciq32.us-east-1.rds.amazonaws.com
RDS_PORT=3306

# CSRF
DJANGO_CSRF_TRUSTED_ORIGINS=https://seu-app.elasticbeanstalk.com,https://seu-dominio.com

# HSTS
DJANGO_SECURE_HSTS_SECONDS=31536000
```

### 2. Inicializar Projeto Beanstalk

```bash
# Navegar para raiz do projeto
cd /caminho/para/RestEB

# Inicializar EB
eb init \
  --platform python-3.12 \
  --region us-east-1 \
  --application catalogo-api

# Responder perguntas interativas:
# - Application name: catalogo-api
# - Platform branch: Python 3.12 running on 64bit Amazon Linux 2023
# - CodeCommit: N
# - SSH Key: N (se primeira vez, deixe criar)
```

### 3. Criar Ambiente Beanstalk

```bash
# Criar ambiente de produção
eb create production \
  --instance-type t3.small \
  --envvars-file .env.prod \
  --elb-type application \
  --single

# Ou para high-availability:
eb create production \
  --instance-type t3.small \
  --min-instances 2 \
  --max-instances 6 \
  --scale-trigger-measure-name CPUUtilization \
  --scale-trigger-unit Percent \
  --scale-trigger-lower 20 \
  --scale-trigger-upper 80

# Monitorar criação
eb events -f
```

### 4. Configurar Variáveis de Ambiente após Criação

```bash
# Se precisar atualizar variáveis depois
eb setenv \
  SECRET_KEY="$(python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())')" \
  DJANGO_DEBUG=False \
  DJANGO_ALLOWED_HOSTS="seu-app.elasticbeanstalk.com"

# Ver variáveis atuais
eb printenv
```

### 5. Configurar RDS Security Group

```bash
# Obter security group do Beanstalk
EB_SECURITY_GROUP=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=awseb-*" \
  --query 'SecurityGroups[0].GroupId' \
  --output text)

# Autorizar acesso ao RDS
aws ec2 authorize-security-group-ingress \
  --group-id $EB_SECURITY_GROUP \
  --protocol tcp \
  --port 3306 \
  --source-group $EB_SECURITY_GROUP

# Ou por Security Group do RDS
RDS_SECURITY_GROUP="sg-xxxxxxxxx"  # Obter do console RDS
aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SECURITY_GROUP \
  --protocol tcp \
  --port 3306 \
  --source-group $EB_SECURITY_GROUP
```

### 6. Deploy Inicial

```bash
# Deploy da aplicação
eb deploy

# Monitorar progresso
eb events -f

# Verificar status
eb status

# Acessar logs
eb logs

# Abrir no navegador
eb open
```

### 7. Executar Migrações

```bash
# SSH na instância
eb ssh

# Dentro da instância
cd /var/app/current
source /var/app/venv/*/bin/activate
python manage.py migrate
python manage.py collectstatic --noinput

# Sair
exit
```

## 📊 Monitoramento

### CloudWatch

```bash
# Ver métricas da aplicação
eb monitoring set-level enhanced

# Abrir console CloudWatch
aws cloudwatch list-metrics \
  --namespace AWS/ElasticBeanstalk \
  --query 'Metrics[*].[MetricName]' \
  --output table
```

### Logs

```bash
# Acompanhar logs em tempo real
eb logs -z

# Baixar logs
eb logs -a

# Limpar logs
eb logs --stream
```

## 🔍 Troubleshooting

### Verificar saúde da aplicação

```bash
# Status detalhado
eb health

# Status em tempo real
eb health --refresh
```

### Diagnosticar problemas

```bash
# Coletar diag bundle
eb diagnostics -b

# Ver erros recentes
eb ssh -c "tail -f /var/log/eb-engine.log"
```

### Rollback de versão

```bash
# Ver versões deployadas
eb appversion

# Fazer rollback
eb abort  # Cancela deploy em andamento
eb deploy --version app-version-id  # Deploy versão específica
```

## 📈 Escalabilidade

### Auto-scaling

```bash
# Ajustar limites de escala
eb scale 3  # Mínimo 3 instâncias

# Configurar eventos de scaling
eb config  # Editar em YAML
```

Dentro da config:
```yaml
AWSEBAutoScalingScaleUpPolicy.json:
  AdjustmentType: ChangeInCapacity
  MetricAggregationType: Average
  ScalingAdjustment: 1
```

## 🔐 Segurança

### HTTPS

```bash
# Conseguir certificado SSL via ACM
aws acm request-certificate \
  --domain-name seu-dominio.com \
  --subject-alternative-names "*.seu-dominio.com" \
  --validation-method DNS

# No console EB:
# 1. Configuration > Load Balancer > HTTPS
# 2. Selecionar certificado ACM
# 3. Aplicar mudanças
```

### Firewall

```bash
# Restringir acesso SSH
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxx \
  --protocol tcp \
  --port 22 \
  --cidr SEU_IP/32
```

## 🧪 Teste Local Antes de Deploy

```bash
# 1. Simular ambiente Beanstalk localmente
export SECRET_KEY=$(python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())')
export DJANGO_DEBUG=False
export DJANGO_ALLOWED_HOSTS=localhost
export RDS_DB_NAME=test_db
export RDS_USERNAME=test_user
export RDS_PASSWORD=test_pass
export RDS_HOSTNAME=localhost

# 2. Executar migrations (se usando SQLite local)
python manage.py migrate

# 3. Rodar Gunicorn
gunicorn catalogo.wsgi:application --workers 4 --bind 0.0.0.0:8000

# 4. Testar health check
curl http://localhost:8000/

# 5. Testar API
curl http://localhost:8000/api/produtos/
```

## 📝 Checklist Pré-Deploy

- [ ] `Procfile` configurado com Gunicorn
- [ ] `runtime.txt` com Python 3.12
- [ ] `.ebextensions/` com todas as configurações
- [ ] `requirements.txt` atualizado
- [ ] `SECRET_KEY` definida no ambiente
- [ ] `DJANGO_ALLOWED_HOSTS` configurado
- [ ] RDS criado e acessível
- [ ] Security groups configurados
- [ ] Migrations testadas localmente
- [ ] Static files coletados: `python manage.py collectstatic`
- [ ] `.gitignore` atualizado
- [ ] Branches limpas (commit antes de deploy)
- [ ] Backup do banco de dados existente (se migração)

## ⚡ Comandos Úteis

```bash
# Ver status de tudo
eb status --verbose

# Atualizar aplicação
eb deploy --staged  # Primeiro commit + push

# SSH rápido
eb ssh

# Abrir console
eb open /admin

# Recriar ambiente
eb terminate production
eb create production

# Conectar ao RDS
mysql -h seu-db.rds.amazonaws.com -u admin -p

# Ver variáveis de ambiente
eb printenv | grep RDS

# Atualizar uma var específica
eb setenv RDS_PASSWORD=nova-senha

# Limpar cache
eb abort  # Cancela deploy
eb setenv DJANGO_DEBUG=False  # Força redeploy
```

## 🔄 Atualizações Futuras

### Migrar para Multi-AZ (Alta Disponibilidade)

```bash
# Backup antes de mudanças
eb clone production --name production-backup

# Configurar Multi-AZ via console ou CLI
eb config  # Editar LoadBalancer para Multi-AZ

# Aplicar mudanças
eb deploy
```

### Adicionar CDN CloudFront

```bash
# Será feito após primeiro deploy bem-sucedido
# Melhor para otimizar distribuição de assets
```

---

**Última atualização:** 2026-05-21  
**Versão do Django:** 6.0.4  
**Python:** 3.12
