# 💻 Rotação de Chaves via AWS CLI

Este documento reúne os comandos para ativar e validar a rotação de chaves KMS pela linha de comando. Os conceitos e justificativas de cada etapa estão documentados nos arquivos da pasta `docs/`.

A chave utilizada (`myKMSKey`) foi criada no [Lab de Criptografia](../01-lab-criptografia/).

---

## Contexto de Execução

Os comandos deste guia rodam em dois ambientes diferentes, refletindo a separação entre administrar a chave e usá-la.

| Operação | Onde executar | Por quê |
|----------|---------------|---------|
| Ativar rotação, rotação sob demanda, consultar status e histórico | CloudShell (usuário administrador) | São operações administrativas da chave |
| Criptografar e descriptografar arquivos | Instância EC2 (role da aplicação) | A role da EC2 tem apenas permissões de uso: Encrypt, Decrypt e GenerateDataKey |

> Essa separação segue o princípio do menor privilégio: a aplicação usa a chave, mas não a administra. Tentar rodar um comando administrativo pela EC2 retorna `AccessDeniedException`.

---

## Pré-requisitos

- AWS CLI configurada (o CloudShell já vem com ela e usa as credenciais do console)
- Região definida como `us-east-1`
- Chave KMS simétrica existente (criada no Lab 01)

---

## Valores a Substituir

Antes de executar, ajuste estes valores conforme o seu ambiente:

| Placeholder | O que é | Como obter |
|-------------|---------|------------|
| `alias/myKMSKey` | Alias da chave criada no Lab 01 | Se você usou outro nome, troque aqui |
| `<INSIRA_O_KEY_ID>` | Identificador da chave, usado onde o alias não é aceito | Comando abaixo |

Para obter o `<INSIRA_O_KEY_ID>` a partir do alias:

```bash
aws kms describe-key \
  --key-id alias/myKMSKey \
  --query KeyMetadata.KeyId \
  --output text \
  --region us-east-1
```

**Saída (exemplo):**

```
5477ae28-3cf6-412a-84e0-b03cfb140a9b
```

> Dica: copie o Key ID para um bloco de notas. Ele é usado em todos os comandos de rotação.

---

## Alias ou Key ID?

Nem todos os comandos KMS tratam alias e Key ID da mesma forma. A família de comandos de rotação atua diretamente sobre o material da chave e exige o Key ID (ou ARN). Usar alias nesses casos retorna `InvalidArnException`.

| Comando | Aceita alias? |
|---------|:-------------:|
| `describe-key` | ✅ Sim |
| `enable-key-rotation` | ❌ Só Key ID |
| `get-key-rotation-status` | ❌ Só Key ID |
| `rotate-key-on-demand` | ❌ Só Key ID |
| `list-key-rotations` | ❌ Só Key ID |

---

## 1. Ativar a Rotação Automática

🔹 **Ambiente:** CloudShell (admin)

```bash
aws kms enable-key-rotation \
  --key-id <INSIRA_O_KEY_ID> \
  --region us-east-1
```

Em caso de sucesso, o comando não retorna saída.

> Por padrão, o período de rotação é de 365 dias. É possível definir um valor entre 90 e 2.560 dias com o parâmetro `--rotation-period-in-days`.

### Verificar o status

```bash
aws kms get-key-rotation-status \
  --key-id <INSIRA_O_KEY_ID> \
  --region us-east-1
```

**Saída:**

```json
{
    "KeyRotationEnabled": true,
    "KeyId": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/5477ae28-3cf6-412a-84e0-b03cfb140a9b",
    "RotationPeriodInDays": 365,
    "NextRotationDate": "2027-08-26T20:28:25.743000+00:00"
}
```

O campo `NextRotationDate` confirma o agendamento da próxima rotação automática (365 dias após a ativação).

---

## 2. Executar Rotação Sob Demanda

🔹 **Ambiente:** CloudShell (admin)

A rotação sob demanda gera novo material criptográfico imediatamente, sem esperar o ciclo automático.

```bash
aws kms rotate-key-on-demand \
  --key-id <INSIRA_O_KEY_ID> \
  --region us-east-1
```

**Saída:**

```json
{
    "KeyId": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/5477ae28-3cf6-412a-84e0-b03cfb140a9b"
}
```

> A AWS permite no máximo 25 rotações sob demanda por chave.

> Cada execução deste comando gera um novo material e o define como atual (`CURRENT`). O material que era atual passa a `NON_CURRENT`. Por isso, ao rodar a rotação mais de uma vez, o material considerado atual muda a cada execução.

---

## 3. Consultar o Histórico de Rotações

🔹 **Ambiente:** CloudShell (admin)

```bash
aws kms list-key-rotations \
  --key-id <INSIRA_O_KEY_ID> \
  --region us-east-1
```

**Saída (após duas rotações sob demanda):**

```json
{
    "Rotations": [
        {
            "KeyId": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/5477ae28-3cf6-412a-84e0-b03cfb140a9b",
            "KeyMaterialId": "fa3ac096f4557eb9728bd8227b80171b2a7cdf7186e0a2f97a9124bebbc5e3cf",
            "KeyMaterialState": "NON_CURRENT",
            "RotationDate": "2026-08-26T20:41:04.621000+00:00",
            "RotationType": "ON_DEMAND"
        },
        {
            "KeyId": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/5477ae28-3cf6-412a-84e0-b03cfb140a9b",
            "KeyMaterialId": "8af5ce798e50ca1bf1c3bbcb88205546330250088d56c7875a449e28aef697e6",
            "KeyMaterialState": "CURRENT",
            "RotationDate": "2026-09-07T19:04:46.252000+00:00",
            "RotationType": "ON_DEMAND"
        }
    ]
}
```

Cada entrada mostra o material gerado, seu estado (`CURRENT` ou `NON_CURRENT`), a data e o tipo da rotação. É essa lista que fornece a evidência de auditoria exigida em ambientes de compliance.

---

## 4. Validar Descriptografia de Dados Antigos

🔹 **Ambiente:** EC2 (role da aplicação)

Confirma que dados criptografados antes da rotação continuam acessíveis. Usa os arquivos gerados no Lab 01 (`datakeyEncrypted.txt`), que estão na instância.

```bash
aws kms decrypt \
  --encryption-context project=practice \
  --ciphertext-blob fileb://datakeyEncrypted.txt \
  --region us-east-1
```

Se a saída retornar o `Plaintext` normalmente, o KMS identificou automaticamente o material anterior à rotação. O detalhamento desse teste, com as evidências, está no doc de [testes](docs/04-testes.md).

---

## 5. Validar Ciclo Completo Pós-Rotação

🔹 **Ambiente:** EC2 (role da aplicação)

Criar arquivo de teste:

```bash
echo "Texto criptografado apos rotacao" > posRotacao.txt
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
Texto criptografado apos rotacao
```

> Nas operações de uso (Encrypt, Decrypt), o alias funciona normalmente. Só a família de comandos de rotação exige o Key ID.

---

## 6. Confirmar que o Key ID Permanece o Mesmo

🔹 **Ambiente:** CloudShell (admin)

```bash
aws kms describe-key \
  --key-id alias/myKMSKey \
  --region us-east-1
```

O `KeyId` e o `Arn` permanecem idênticos aos de antes da rotação. A rotação troca o material interno, mas preserva a identidade da chave, então nenhuma alteração no código das aplicações é necessária.
