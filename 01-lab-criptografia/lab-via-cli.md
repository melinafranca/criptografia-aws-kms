# Laboratório Completo via AWS CLI

Este documento reúne todos os comandos necessários para reproduzir o laboratório de criptografia inteiramente pela linha de comando. Os conceitos e justificativas de cada etapa estão documentados nos arquivos da pasta `docs/`.

---

## Contexto de Execução

Os comandos deste guia rodam em dois ambientes diferentes, refletindo a separação entre provisionar/administrar os recursos e usá-los.

| Etapa | Onde executar | Por quê |
|-------|---------------|---------|
| 1 a 4 (criar role, chave, role da EC2, instância) | CloudShell ou máquina local (usuário administrador) | São operações de provisionamento e administração |
| 5 a 8 (envelope encryption, criptografia direta, testes) | Instância EC2 (role da aplicação) | Usam a chave via role restrita, de dentro da instância |

> A separação segue o princípio do menor privilégio: a instância usa a chave, mas não a administra. Um comando administrativo executado pela EC2 retorna `AccessDeniedException` (ver seção 8).

---

## Pré-requisitos

- AWS CLI configurada com credenciais de um usuário IAM com permissões de administrador (o CloudShell já usa as credenciais do console)
- Região definida como `us-east-1`

```bash
aws configure
```

---

## 1. Criar a Role de Administração do KMS

🔹 **Ambiente:** CloudShell / máquina local (admin)

### Trust policy

Salvar como `trust-policy-account.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Criar a role

```bash
aws iam create-role \
  --role-name KMSKeyAdminRole \
  --assume-role-policy-document file://trust-policy-account.json \
  --description "Role para administracao de chaves KMS no lab de criptografia"
```

### Criar e anexar a política inline

```bash
aws iam put-role-policy \
  --role-name KMSKeyAdminRole \
  --policy-name KMSKeyAdminPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "PermitirAdministracaoChavesKMS",
        "Effect": "Allow",
        "Action": [
          "kms:Create*",
          "kms:Describe*",
          "kms:Enable*",
          "kms:List*",
          "kms:Put*",
          "kms:Update*",
          "kms:Revoke*",
          "kms:Disable*",
          "kms:Get*",
          "kms:Delete*",
          "kms:TagResource",
          "kms:UntagResource",
          "kms:ScheduleKeyDeletion",
          "kms:CancelKeyDeletion",
          "kms:RotateKeyOnDemand"
        ],
        "Resource": "*"
      }
    ]
  }'
```

### Verificar

```bash
aws iam get-role --role-name KMSKeyAdminRole
```

---

## 2. Criar a Chave KMS

🔹 **Ambiente:** CloudShell / máquina local (admin)

### Criar a chave

```bash
aws kms create-key \
  --description "Chave simetrica para criptografia de dados em repouso (lab)" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --region us-east-1
```

**Saída (resumida):**

```json
{
    "KeyMetadata": {
        "KeyId": "5477ae28-3cf6-412a-84e0-b03cfb140a9b",
        "Arn": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/5477ae28-3cf6-412a-84e0-b03cfb140a9b",
        "KeyState": "Enabled",
        "KeySpec": "SYMMETRIC_DEFAULT",
        "KeyUsage": "ENCRYPT_DECRYPT"
    }
}
```

> Copie o `KeyId` retornado para usar nos próximos comandos.

### Criar o alias

O alias permite referenciar a chave como `alias/myKMSKey` em vez do Key ID completo, tornando os comandos mais legíveis.

```bash
aws kms create-alias \
  --alias-name alias/myKMSKey \
  --target-key-id <INSIRA_O_KEY_ID> \
  --region us-east-1
```

### Verificar

```bash
aws kms describe-key --key-id alias/myKMSKey --region us-east-1
```

---

## 3. Criar a Role da EC2

🔹 **Ambiente:** CloudShell / máquina local (admin)

### Trust policy para EC2

```bash
aws iam create-role \
  --role-name EC2KMSLabRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }' \
  --description "Role para instancia EC2 do lab de criptografia com KMS"
```

### Anexar política do Session Manager

```bash
aws iam attach-role-policy \
  --role-name EC2KMSLabRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

### Adicionar política inline de KMS

```bash
aws iam put-role-policy \
  --role-name EC2KMSLabRole \
  --policy-name KMSLabEncryptPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "PermitirOperacoesCriptograficas",
        "Effect": "Allow",
        "Action": [
          "kms:GenerateDataKey",
          "kms:Encrypt",
          "kms:Decrypt"
        ],
        "Resource": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/<INSIRA_O_KEY_ID>"
      }
    ]
  }'
```

### Criar Instance Profile

```bash
aws iam create-instance-profile --instance-profile-name EC2KMSLabProfile
```

```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2KMSLabProfile \
  --role-name EC2KMSLabRole
```

---

## 4. Criar a Instância EC2

🔹 **Ambiente:** CloudShell / máquina local (admin)

### Criar Security Group

```bash
aws ec2 create-security-group \
  --group-name lab-criptografia \
  --description "Security group para lab de criptografia - sem regras de entrada" \
  --region us-east-1
```

> Copie o `GroupId` retornado para usar no próximo comando.

### Executar a instância

```bash
aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t2.micro \
  --iam-instance-profile Name=EC2KMSLabProfile \
  --security-group-ids <INSIRA_O_SG_ID> \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ec2-lab-criptografia}]' \
  --region us-east-1
```

> Em vez de fixar um ID de AMI (que muda com o tempo e varia por região), a sintaxe `resolve:ssm:` busca automaticamente a AMI mais recente do Amazon Linux 2023 no momento da execução. Assim o comando não desatualiza.

> Substituir `<INSIRA_O_SG_ID>` pelo ID do security group criado no passo anterior. Copie o `InstanceId` retornado.

### Conectar via Session Manager

Aguardar a instância ficar em estado `running`:

```bash
aws ec2 describe-instances \
  --instance-ids <INSIRA_O_INSTANCE_ID> \
  --query "Reservations[].Instances[].State.Name" \
  --region us-east-1
```

Conectar:

```bash
aws ssm start-session --target <INSIRA_O_INSTANCE_ID> --region us-east-1
```

Após conectar:

```bash
sudo su ec2-user
```

```bash
cd ~
```

---

## 5. Envelope Encryption

🔹 **Ambiente:** instância EC2 (role da aplicação), conectada via Session Manager na etapa anterior

### Criar arquivo com dados sensíveis

```bash
echo "Texto secreto para criptografar." > samplesecret.txt
```

### Gerar a Data Key

```bash
aws kms generate-data-key \
  --key-id alias/myKMSKey \
  --key-spec AES_256 \
  --encryption-context project=practice \
  --region us-east-1
```

> Copie os valores de `Plaintext` e `CiphertextBlob` para um bloco de notas.

### Salvar as chaves em arquivos

```bash
echo 'VALOR_PLAINTEXT' | base64 --decode > datakeyPlainText.txt
```

```bash
echo 'VALOR_CIPHERTEXTBLOB' | base64 --decode > datakeyEncrypted.txt
```

### Criptografar o arquivo

```bash
openssl enc -e -aes256 -pbkdf2 \
  -in samplesecret.txt \
  -out encryptedSecret.txt \
  -k fileb://datakeyPlainText.txt
```

### Verificar a criptografia

```bash
more encryptedSecret.txt
```

### Excluir a plaintext data key

```bash
rm datakeyPlainText.txt
```

### Recuperar a data key

```bash
aws kms decrypt \
  --encryption-context project=practice \
  --ciphertext-blob fileb://datakeyEncrypted.txt \
  --region us-east-1
```

> Copie o valor de `Plaintext` retornado.

```bash
echo 'VALOR_PLAINTEXT' | base64 --decode > datakeyPlainText.txt
```

### Descriptografar o arquivo

```bash
openssl enc -d -aes256 -pbkdf2 \
  -in encryptedSecret.txt \
  -k fileb://datakeyPlainText.txt
```

**Saída esperada:**
```
Texto secreto para criptografar.
```

---

## 6. Criptografia Direta

🔹 **Ambiente:** instância EC2 (role da aplicação)

### Criar arquivo

```bash
echo "Arquivo secreto: criptografia direta sem data key." > NewSecretFile.txt
```

### Criptografar

```bash
aws kms encrypt \
  --key-id alias/myKMSKey \
  --plaintext fileb://NewSecretFile.txt \
  --encryption-context project=practice \
  --output text \
  --query CiphertextBlob \
  --region us-east-1 | base64 --decode > NewSecretsEncryptedFile.txt
```

### Verificar

```bash
cat NewSecretsEncryptedFile.txt
```

### Descriptografar

```bash
aws kms decrypt \
  --ciphertext-blob fileb://NewSecretsEncryptedFile.txt \
  --encryption-context project=practice \
  --output text \
  --query Plaintext \
  --region us-east-1 | base64 --decode > NewSecretsDecryptedFile.txt
```

```bash
cat NewSecretsDecryptedFile.txt
```

**Saída esperada:**
```
Arquivo secreto: criptografia direta sem data key.
```

---

## 7. Testar Encryption Context

🔹 **Ambiente:** instância EC2 (role da aplicação)

### Sem context (deve falhar)

```bash
aws kms decrypt \
  --ciphertext-blob fileb://NewSecretsEncryptedFile.txt \
  --output text \
  --query Plaintext \
  --region us-east-1
```

### Com context errado (deve falhar)

```bash
aws kms decrypt \
  --ciphertext-blob fileb://NewSecretsEncryptedFile.txt \
  --encryption-context project=errado \
  --output text \
  --query Plaintext \
  --region us-east-1
```

### Com context correto (deve funcionar)

```bash
aws kms decrypt \
  --ciphertext-blob fileb://NewSecretsEncryptedFile.txt \
  --encryption-context project=practice \
  --output text \
  --query Plaintext \
  --region us-east-1
```

---

## 8. Teste de Menor Privilégio

🔹 **Ambiente:** instância EC2 (role da aplicação)

Tentar uma ação não permitida pela role da EC2:

```bash
aws kms schedule-key-deletion --key-id <INSIRA_O_KEY_ID> --region us-east-1
```

> Substituir `<INSIRA_O_KEY_ID>` pelo identificador da chave `myKMSKey`. Esse comando não aceita alias.

**Resultado esperado:** `AccessDeniedException`
