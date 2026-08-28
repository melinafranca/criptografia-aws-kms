# 💻 Rotação de Chaves via AWS CLI

Este documento reúne todos os comandos necessários para ativar e validar a rotação de chaves KMS inteiramente pela linha de comando. Os conceitos e justificativas de cada etapa estão documentados nos arquivos da pasta `docs/`.

A chave utilizada (`myKMSKey`) foi criada no [Lab de Criptografia](../01-lab-criptografia/).

---

## Pré-requisitos

- AWS CLI configurada com credenciais de um usuário IAM com permissões para operar o KMS
- Região definida como `us-east-1`
- Chave KMS simétrica existente (`alias/myKMSKey`)
- Permissões `kms:EnableKeyRotation`, `kms:GetKeyRotationStatus` e `kms:RotateKeyOnDemand`

```bash
aws configure
```

---

## 1. Ativar a Rotação Automática

```bash
aws kms enable-key-rotation \
  --key-id alias/myKMSKey \
  --region us-east-1
```

> Por padrão, o período de rotação é de 365 dias. É possível definir um valor entre 90 e 2.560 dias com o parâmetro `--rotation-period-in-days`.

### Verificar o status

```bash
aws kms get-key-rotation-status \
  --key-id alias/myKMSKey \
  --region us-east-1
```

**Saída esperada:**

```json
{
  "KeyRotationEnabled": true
}
```

---

## 2. Executar Rotação Sob Demanda

A rotação sob demanda gera novo material criptográfico imediatamente, sem esperar o ciclo automático.

```bash
aws kms rotate-key-on-demand \
  --key-id alias/myKMSKey \
  --region us-east-1
```

**Saída esperada:**

```json
{
  "KeyId": "arn:aws:kms:us-east-1:ACCOUNT_ID:key/KEY-ID"
}
```

> A AWS permite no máximo 25 rotações sob demanda por chave.

---

## 3. Consultar o Histórico de Rotações

> Diferente da maioria dos comandos KMS, o `list-key-rotations` **não aceita alias**. É preciso informar o Key ID (ou o ARN) da chave. Usar o alias retorna `InvalidArnException`.

```bash
aws kms list-key-rotations \
  --key-id KEY-ID \
  --region us-east-1
```

> Substituir `KEY-ID` pelo identificador da chave (ex: `5477ae28-3cf6-412a-84e0-b03cfb140a9b`).

Esse comando mostra todas as rotações (automáticas e sob demanda) realizadas na chave, com data e tipo de cada uma.

---

## 4. Validar Descriptografia de Dados Antigos

Confirmar que dados criptografados antes da rotação continuam acessíveis. Usar os arquivos gerados no Lab 01 (`datakeyEncrypted.txt`):

```bash
aws kms decrypt \
  --encryption-context project=practice \
  --ciphertext-blob fileb://datakeyEncrypted.txt \
  --region us-east-1
```

Se a saída retornar o `Plaintext` normalmente, o KMS identificou automaticamente o material criptográfico correto (versão anterior à rotação).

---

## 5. Validar Ciclo Completo Pós-Rotação

Criar arquivo de teste:

```bash
echo "Texto criptografado após rotação" > posRotacao.txt
```

Criptografar (usará o material mais recente):

```bash
aws kms encrypt \
  --key-id alias/myKMSKey \
  --plaintext fileb://posRotacao.txt \
  --encryption-context project=practice \
  --output text --query CiphertextBlob \
  --region us-east-1 | base64 --decode > posRotacaoEncrypted.txt
```

Descriptografar:

```bash
aws kms decrypt \
  --ciphertext-blob fileb://posRotacaoEncrypted.txt \
  --encryption-context project=practice \
  --output text --query Plaintext \
  --region us-east-1 | base64 --decode
```

**Saída esperada:**

```
Texto criptografado após rotação
```

---

## 6. Confirmar que o Key ID Permanece o Mesmo

```bash
aws kms describe-key \
  --key-id alias/myKMSKey \
  --region us-east-1
```

O `KeyId` e o `Arn` devem ser idênticos aos de antes da rotação. Nenhuma alteração no código das aplicações é necessária.
