# 💻 Monitoramento de Rotação via AWS CLI

Este documento reproduz o laboratório pela linha de comando. Os conceitos e a comparação entre os dois caminhos estão nos arquivos da pasta `docs/`.

Todos os comandos rodam no **AWS CloudShell**, com as credenciais do usuário administrador. A região é `us-east-1`.

> Este guia foi executado de ponta a ponta. A rotação sob demanda disparou os dois caminhos e as duas notificações por email chegaram, confirmando que o fluxo funciona também pela CLI. Reproduzir pela linha de comando expõe passos que o console automatiza (a policy do bucket, a role da trilha para o CloudWatch e a policy do tópico SNS), o que ajuda a entender o que acontece nos bastidores.

---

## Valores a Substituir

| Placeholder | O que é | Como obter |
|-------------|---------|------------|
| `<INSIRA_O_KEY_ID>` | Key ID da chave criada | Passo 1 |
| `<ACCOUNT_ID>` | ID da conta AWS | Aparece nos ARNs |
| `<EMAIL>` | Email que receberá os alertas | Seu email |

---

## 1. Criar a Chave e Ativar a Rotação

Criar a chave dedicada ao monitoramento:

```bash
aws kms create-key \
  --description "Chave para o laboratório de monitoramento de rotação" \
  --region us-east-1
```

O retorno traz o bloco `KeyMetadata`, com o `KeyId` (formato UUID) e `MultiRegion: false`. Guardar o `KeyId` para os próximos comandos.

Criar o alias:

```bash
aws kms create-alias \
  --alias-name alias/rotationMonitorKey \
  --target-key-id <INSIRA_O_KEY_ID> \
  --region us-east-1
```

Ativar a rotação automática (período de 90 dias, o mínimo permitido):

```bash
aws kms enable-key-rotation \
  --key-id <INSIRA_O_KEY_ID> \
  --rotation-period-in-days 90 \
  --region us-east-1
```

Os comandos `create-alias` e `enable-key-rotation` não retornam saída em caso de sucesso.

> A rotação automática só dispara a cada 90 dias, então não é testável no tempo do lab. O gatilho dos testes é a rotação sob demanda (passo 5).

---

## 2. Criar a Trilha do CloudTrail

A trilha é a fundação dos dois caminhos. Sem ela, os eventos não são entregues ao EventBridge nem ao CloudWatch. Pelo console, esta etapa é um formulário; pela CLI, expõe cada peça que o console monta automaticamente.

### 2.1. Bucket S3 e sua policy

Criar o bucket que armazena os logs (o nome é único globalmente):

```bash
aws s3 mb s3://kms-rotation-trail-logs-<ACCOUNT_ID> --region us-east-1
```

O CloudTrail precisa de permissão explícita para gravar no bucket. Criar o arquivo de policy:

```bash
cat > bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::kms-rotation-trail-logs-<ACCOUNT_ID>"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::kms-rotation-trail-logs-<ACCOUNT_ID>/AWSLogs/<ACCOUNT_ID>/*",
      "Condition": {"StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}}
    }
  ]
}
EOF
```

Aplicar a policy:

```bash
aws s3api put-bucket-policy --bucket kms-rotation-trail-logs-<ACCOUNT_ID> --policy file://bucket-policy.json
```

### 2.2. Criar a trilha e iniciar o logging

```bash
aws cloudtrail create-trail \
  --name kms-rotation-trail \
  --s3-bucket-name kms-rotation-trail-logs-<ACCOUNT_ID> \
  --region us-east-1
```

```bash
aws cloudtrail start-logging --name kms-rotation-trail --region us-east-1
```

> Criar a trilha não a coloca para rodar. O `start-logging` é um passo separado e obrigatório, que o console executa automaticamente. Pela CLI, a trilha nasce parada. A trilha criada por CLI também vem como região única e sem validação de log, enquanto o console cria multi-região com validação por padrão.

### 2.3. Conectar a trilha ao CloudWatch Logs

Esta é a parte que o Caminho 2 exige. Envolve criar o Log Group, uma role que o CloudTrail assume e o vínculo entre eles.

Criar o Log Group:

```bash
aws logs create-log-group --log-group-name aws-cloudtrail-logs-kms-rotation --region us-east-1
```

Criar a role que o CloudTrail assume para gravar no Log Group:

```bash
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

```bash
aws iam create-role \
  --role-name CloudTrailRoleForCloudWatchLogs-kms-rotation \
  --assume-role-policy-document file://trust-policy.json
```

Dar à role permissão de escrever no Log Group:

```bash
cat > role-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:us-east-1:<ACCOUNT_ID>:log-group:aws-cloudtrail-logs-kms-rotation:*"
    }
  ]
}
EOF
```

```bash
aws iam put-role-policy \
  --role-name CloudTrailRoleForCloudWatchLogs-kms-rotation \
  --policy-name cloudtrail-cwlogs-policy \
  --policy-document file://role-policy.json
```

Vincular o Log Group e a role à trilha:

```bash
aws cloudtrail update-trail \
  --name kms-rotation-trail \
  --cloud-watch-logs-log-group-arn "arn:aws:logs:us-east-1:<ACCOUNT_ID>:log-group:aws-cloudtrail-logs-kms-rotation:*" \
  --cloud-watch-logs-role-arn "arn:aws:iam::<ACCOUNT_ID>:role/CloudTrailRoleForCloudWatchLogs-kms-rotation" \
  --region us-east-1
```

No retorno, os campos `CloudWatchLogsLogGroupArn` e `CloudWatchLogsRoleArn` aparecem preenchidos, confirmando o vínculo.

> Estes sete comandos (bucket, policy do bucket, trilha, logging, Log Group, role, vínculo) são o que o console resolve em um único formulário. A CLI expõe cada permissão e cada ligação, o que ajuda a entender o que realmente acontece nos bastidores. Se o `update-trail` falhar logo após criar a role, aguarde alguns segundos: a role IAM leva um instante para propagar.

---

## 3. Caminho 1: EventBridge + SNS

### 3.1. Tópico SNS e assinatura

Criar o tópico:

```bash
aws sns create-topic --name kms-rotation-alerts --region us-east-1
```

O retorno traz o `TopicArn`. Assinar o tópico por email:

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:<ACCOUNT_ID>:kms-rotation-alerts \
  --protocol email \
  --notification-endpoint <EMAIL> \
  --region us-east-1
```

O retorno traz `SubscriptionArn: "pending confirmation"`. A AWS envia um email com um link de confirmação, que precisa ser aceito antes de qualquer notificação chegar.

> A confirmação é feita clicando no link do email. Ela não pode ser automatizada com um comando simples: o link contém um token único, enviado só para o email, que prova o consentimento do dono do endereço. Existe o `aws sns confirm-subscription --token`, mas ele exige esse token extraído da URL do email, então, para uso manual, clicar no link é mais direto.

### 3.2. Regra do EventBridge

Criar a regra filtrando o evento de rotação:

```bash
aws events put-rule \
  --name kms-rotation-monitor-rule \
  --event-pattern '{"source":["aws.kms"],"detail-type":["AWS API Call via CloudTrail"],"detail":{"eventName":["RotateKeyOnDemand"]}}' \
  --region us-east-1
```

Apontar a regra para o tópico SNS:

```bash
aws events put-targets \
  --rule kms-rotation-monitor-rule \
  --targets "Id"="1","Arn"="arn:aws:sns:us-east-1:<ACCOUNT_ID>:kms-rotation-alerts" \
  --region us-east-1
```

> O `FailedEntryCount: 0` confirma que o alvo foi vinculado, mas não que o EventBridge pode publicar no tópico. Essa permissão é o passo 3.3.

### 3.3. Permitir que o EventBridge publique no SNS

Pelo console, essa permissão é configurada automaticamente ao definir o destino. Pela CLI, é preciso ajustar a policy do tópico.

```bash
cat > sns-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEventBridgePublish",
      "Effect": "Allow",
      "Principal": {"Service": "events.amazonaws.com"},
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:us-east-1:<ACCOUNT_ID>:kms-rotation-alerts"
    }
  ]
}
EOF
```

```bash
aws sns set-topic-attributes \
  --topic-arn arn:aws:sns:us-east-1:<ACCOUNT_ID>:kms-rotation-alerts \
  --attribute-name Policy \
  --attribute-value file://sns-policy.json \
  --region us-east-1
```

> A policy acima permite que qualquer regra do EventBridge da conta publique no tópico. Para aplicar o menor privilégio, dá para restringir à regra específica com uma `Condition`, comparando o `aws:SourceArn` com o ARN da regra:
>
> ```json
> "Condition": {
>   "ArnEquals": {
>     "aws:SourceArn": "arn:aws:events:us-east-1:<ACCOUNT_ID>:rule/kms-rotation-monitor-rule"
>   }
> }
> ```

---

## 4. Caminho 2: CloudWatch Metric Filter + Alarme

Criar o metric filter no Log Group da trilha:

```bash
aws logs put-metric-filter \
  --log-group-name aws-cloudtrail-logs-kms-rotation \
  --filter-name kms-rotation-filter \
  --filter-pattern '{ $.eventName = "RotateKeyOnDemand" }' \
  --metric-transformations \
      metricName=KMSKeyRotationCount,metricNamespace=KMSMonitoring,metricValue=1,defaultValue=0 \
  --region us-east-1
```

> O `defaultValue=0` faz a métrica reportar uma série contínua, mais estável para o alarme. Sem dimensões, para não anular esse efeito.

Criar o alarme baseado na métrica:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name kms-rotation-alarm \
  --alarm-description "Dispara na rotação da rotationMonitorKey" \
  --namespace KMSMonitoring \
  --metric-name KMSKeyRotationCount \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:<ACCOUNT_ID>:kms-rotation-alerts \
  --region us-east-1
```

Nenhum dos dois comandos retorna saída em caso de sucesso. Confirmar que o alarme foi criado:

```bash
aws cloudwatch describe-alarms \
  --alarm-names kms-rotation-alarm \
  --query "MetricAlarms[0].StateValue" \
  --output text \
  --region us-east-1
```

O estado começa em `INSUFFICIENT_DATA` e passa a `OK` quando a métrica reporta 0.

---

## 5. Testar: Disparar a Rotação

A rotação sob demanda exige o Key ID, não aceita alias:

```bash
aws kms rotate-key-on-demand --key-id <INSIRA_O_KEY_ID> --region us-east-1
```

Após alguns minutos (latência do CloudTrail), os dois caminhos reagem:

- **Caminho 1:** email do EventBridge, quase imediato
- **Caminho 2:** o alarme muda para `ALARM` e envia o email (mais lento, por causa da agregação em métrica)

Confirmar que o evento foi registrado no Log Group:

```bash
aws logs filter-log-events \
  --log-group-name aws-cloudtrail-logs-kms-rotation \
  --filter-pattern '{ $.eventName = "RotateKeyOnDemand" }' \
  --region us-east-1
```

Verificar o estado do alarme (deve estar em `ALARM` após a rotação):

```bash
aws cloudwatch describe-alarms \
  --alarm-names kms-rotation-alarm \
  --query "MetricAlarms[0].StateValue" \
  --output text \
  --region us-east-1
```

---

## Referências

- [rotate-key-on-demand (CLI)](https://docs.aws.amazon.com/cli/latest/reference/kms/rotate-key-on-demand.html)
- [put-metric-filter (CLI)](https://docs.aws.amazon.com/cli/latest/reference/logs/put-metric-filter.html)
- [put-metric-alarm (CLI)](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-alarm.html)
