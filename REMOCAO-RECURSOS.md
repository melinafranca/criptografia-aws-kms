# Remoção de Recursos

Este documento orienta a remoção de todos os recursos criados ao longo dos laboratórios deste repositório. Siga a ordem indicada para evitar erros de dependência.

> Só remova os recursos quando tiver concluído todos os projetos que deseja executar. A chave `myKMSKey`, a instância EC2 e as roles são compartilhadas entre os projetos 01 (Criptografia) e 02 (Rotação de Chaves). Os projetos 03 (Multi-Região) e 04 (Monitoramento de Rotação) criam recursos próprios e independentes, então podem ser removidos separadamente, sem afetar os demais.

---

## Ordem de Remoção

A ordem importa. Alguns recursos dependem de outros para serem removidos corretamente.

---

### 1. Encerrar a Instância EC2

1. No console EC2, selecionar a instância `ec2-lab-criptografia`
2. Clicar em **Estado da instância → Encerrar instância**
3. Confirmar a ação

Via CLI:

```bash
aws ec2 terminate-instances --instance-ids <INSIRA_O_INSTANCE_ID> --region us-east-1
```

---

### 2. Remover o Security Group

Aguardar a instância ficar com estado **Encerrada** antes de remover o security group.

1. No console EC2, ir em **Rede e segurança → Security groups**
2. Selecionar `lab-criptografia`
3. Clicar em **Ações → Excluir security groups**

Via CLI:

```bash
aws ec2 delete-security-group --group-name lab-criptografia --region us-east-1
```

---

### 3. Agendar a Exclusão da Chave KMS

O KMS não permite exclusão imediata. A chave entra em um período de espera (mínimo 7 dias, máximo 30 dias) antes de ser deletada permanentemente.

1. No console KMS, selecionar a chave `myKMSKey`
2. Clicar em **Ações da chave → Agendar exclusão de chave**
3. Definir o período de espera (7 dias é o mínimo)
4. Confirmar

Via CLI:

```bash
aws kms schedule-key-deletion \
  --key-id <INSIRA_O_KEY_ID> \
  --pending-window-in-days 7 \
  --region us-east-1
```

> O `schedule-key-deletion` não aceita alias. Use o Key ID (ou o ARN) da chave. Substitua `<INSIRA_O_KEY_ID>` pelo identificador de `myKMSKey`.

> Durante o período de espera, a chave fica desativada (não pode ser usada para criptografia). Se mudar de ideia, é possível cancelar com `aws kms cancel-key-deletion`.

---

### 4. Chave Multi-Região (Projeto 03)

Remover primeiro a réplica, depois a primária (o `schedule-key-deletion` exige o Key ID de cada região, não o alias):

```bash
# Agendar exclusão da réplica
aws kms schedule-key-deletion \
  --key-id <INSIRA_O_REPLICA_KEY_ID> \
  --pending-window-in-days 7 \
  --region eu-west-1

# Agendar exclusão da primária
aws kms schedule-key-deletion \
  --key-id <INSIRA_O_PRIMARY_KEY_ID> \
  --pending-window-in-days 7 \
  --region us-east-1
```

> O alias `myMultiRegionKey` existe em cada região. Se a chave já foi agendada para exclusão, ele é removido junto. Para remover antes, use `aws kms delete-alias` em cada região.

---

### 5. Recursos do Monitoramento de Rotação (Projeto 04)

Este projeto cria vários recursos encadeados. Remova na ordem abaixo, do consumidor para a fonte, para evitar erros de dependência.

**Alarme do CloudWatch:**

```bash
aws cloudwatch delete-alarms --alarm-names kms-rotation-alarm --region us-east-1
```

**Metric filter:**

```bash
aws logs delete-metric-filter \
  --log-group-name aws-cloudtrail-logs-kms-rotation \
  --filter-name kms-rotation-filter \
  --region us-east-1
```

**Regra do EventBridge** (remover o alvo antes da regra):

```bash
aws events remove-targets --rule kms-rotation-monitor-rule --ids "1" --region us-east-1
aws events delete-rule --name kms-rotation-monitor-rule --region us-east-1
```

**Tópico SNS** (remove o tópico e suas assinaturas):

```bash
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:<ACCOUNT_ID>:kms-rotation-alerts --region us-east-1
```

**Trilha do CloudTrail** (parar o logging antes de excluir):

```bash
aws cloudtrail stop-logging --name kms-rotation-trail --region us-east-1
aws cloudtrail delete-trail --name kms-rotation-trail --region us-east-1
```

**Role do CloudTrail para o CloudWatch Logs:**

```bash
aws iam delete-role-policy \
  --role-name CloudTrailRoleForCloudWatchLogs-kms-rotation \
  --policy-name cloudtrail-cwlogs-policy
aws iam delete-role --role-name CloudTrailRoleForCloudWatchLogs-kms-rotation
```

**Log Group do CloudWatch:**

```bash
aws logs delete-log-group --log-group-name aws-cloudtrail-logs-kms-rotation --region us-east-1
```

**Bucket S3** (esvaziar antes de excluir):

```bash
aws s3 rm s3://kms-rotation-trail-logs-<ACCOUNT_ID> --recursive
aws s3 rb s3://kms-rotation-trail-logs-<ACCOUNT_ID>
```

**Chave `rotationMonitorKey`** (agendar exclusão pelo Key ID):

```bash
aws kms schedule-key-deletion \
  --key-id <INSIRA_O_KEY_ID> \
  --pending-window-in-days 7 \
  --region us-east-1
```

---

### 6. Remover a Role da EC2

Antes de deletar a role, é necessário remover as políticas anexadas e o instance profile.

```bash
# Remover política inline
aws iam delete-role-policy \
  --role-name EC2KMSLabRole \
  --policy-name KMSLabEncryptPolicy

# Desanexar política gerenciada
aws iam detach-role-policy \
  --role-name EC2KMSLabRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Remover role do instance profile
aws iam remove-role-from-instance-profile \
  --instance-profile-name EC2KMSLabProfile \
  --role-name EC2KMSLabRole

# Deletar instance profile
aws iam delete-instance-profile \
  --instance-profile-name EC2KMSLabProfile

# Deletar a role
aws iam delete-role --role-name EC2KMSLabRole
```

---

### 7. Remover a Role Admin do KMS

```bash
# Remover política inline
aws iam delete-role-policy \
  --role-name KMSKeyAdminRole \
  --policy-name KMSKeyAdminPolicy

# Deletar a role
aws iam delete-role --role-name KMSKeyAdminRole
```

---

### 8. Remover o Alias da Chave (opcional)

Se a chave já foi agendada para exclusão, o alias será removido automaticamente após o período de espera. Para remover antes:

```bash
aws kms delete-alias --alias-name alias/myKMSKey --region us-east-1
```

---

## Verificação Final

Após a remoção, confirmar que não restam recursos:

```bash
# Verificar instâncias
aws ec2 describe-instances --filters "Name=tag:Name,Values=ec2-lab-criptografia" --query "Reservations[].Instances[].State.Name" --region us-east-1

# Verificar roles
aws iam get-role --role-name EC2KMSLabRole 2>&1
aws iam get-role --role-name KMSKeyAdminRole 2>&1

# Verificar chaves KMS
aws kms describe-key --key-id alias/myKMSKey --region us-east-1
aws kms describe-key --key-id alias/rotationMonitorKey --region us-east-1

# Verificar recursos do monitoramento (Projeto 04)
aws cloudtrail describe-trails --query "trailList[?Name=='kms-rotation-trail']" --region us-east-1
aws cloudwatch describe-alarms --alarm-names kms-rotation-alarm --region us-east-1
```

Se todos retornarem erros (`NoSuchEntity`, `NotFoundException`) ou listas vazias, a remoção foi concluída.

---

## Custo Residual

Após agendar a exclusão de uma chave KMS, ela para de gerar custo imediatamente (chaves em estado `PendingDeletion` não são cobradas). A instância EC2 para de gerar custo assim que é encerrada.

No Projeto 04, os recursos que geram custo contínuo são o bucket S3 e o Log Group do CloudWatch (armazenamento dos logs). Removê-los encerra esse custo. A trilha do CloudTrail, o EventBridge e o SNS não têm custo relevante para o volume de um laboratório.
