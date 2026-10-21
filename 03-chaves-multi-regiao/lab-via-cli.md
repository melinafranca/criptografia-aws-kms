# 💻 Chaves Multi-Região via AWS CLI

Este documento reúne os comandos para criar, replicar e usar uma chave multi-região KMS pela linha de comando. Os conceitos e justificativas de cada etapa estão documentados nos arquivos da pasta `docs/`.

O laboratório usa duas regiões: `us-east-1` (Virgínia do Norte) como primária e `eu-west-1` (Irlanda) como réplica.

---

## Contexto de Execução

Todos os comandos rodam no **AWS CloudShell**, com as credenciais do usuário administrador do console.

| Operação | Onde executar | Por quê |
|----------|---------------|---------|
| Criar, replicar e administrar as chaves | CloudShell (usuário administrador) | São operações administrativas do KMS |
| Criptografar e descriptografar arquivos | CloudShell (usuário administrador) | Neste lab, o mesmo usuário administra e usa as chaves |

> Optei pelo CloudShell em vez da instância EC2 dos labs anteriores por dois motivos: este laboratório é independente (não exige a EC2 nem a role do Lab 01) e a demonstração roda em duas regiões, bastando alterar o parâmetro `--region` para alternar entre elas.

---

## Pré-requisitos

- AWS CLI configurada (o CloudShell já vem com ela e usa as credenciais do console)
- Permissões `kms:CreateKey`, `kms:ReplicateKey`, `kms:CreateAlias`, `kms:Encrypt` e `kms:Decrypt`
- Acesso às regiões `us-east-1` e `eu-west-1`

---

## Valores a Substituir

Antes de executar, ajuste estes valores conforme o seu ambiente:

| Placeholder | O que é | Como obter |
|-------------|---------|------------|
| `<INSIRA_O_KEY_ID>` | Key ID da chave multi-região (prefixo `mrk-`) | Retornado no passo 1 |
| `<ACCOUNT_ID>` | ID da sua conta AWS | Aparece nos ARNs das saídas |

> Dica: copie o Key ID para um bloco de notas assim que criar a chave. Ele é usado em quase todos os comandos seguintes.

---

## 1. Criar a Chave Primária Multi-Região

🔹 **Ambiente:** CloudShell (admin), região `us-east-1`

O parâmetro `--multi-region` é o que diferencia essa chave. Sem ele, o KMS cria uma chave de região única.

```bash
aws kms create-key \
  --description "Chave multi-região para Disaster Recovery" \
  --multi-region \
  --region us-east-1
```

**Saída (resumida):**

```json
{
    "KeyMetadata": {
        "AWSAccountId": "<ACCOUNT_ID>",
        "KeyId": "mrk-1234abcd5678efgh9012ijkl3456mnop",
        "Arn": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/mrk-1234abcd5678efgh9012ijkl3456mnop",
        "Enabled": true,
        "KeyState": "Enabled",
        "KeySpec": "SYMMETRIC_DEFAULT",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "MultiRegion": true,
        "MultiRegionConfiguration": {
            "MultiRegionKeyType": "PRIMARY",
            "PrimaryKey": {
                "Arn": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/mrk-1234abcd5678efgh9012ijkl3456mnop",
                "Region": "us-east-1"
            },
            "ReplicaKeys": []
        }
    }
}
```

> O Key ID vem com o prefixo `mrk-`, que identifica chaves multi-região. O campo `ReplicaKeys` está vazio por enquanto: será preenchido após a replicação (passo 3). Copie o Key ID para substituir `<INSIRA_O_KEY_ID>` nos próximos comandos.

---

## 2. Criar o Alias na Região Primária

🔹 **Ambiente:** CloudShell (admin), região `us-east-1`

```bash
aws kms create-alias \
  --alias-name alias/myMultiRegionKey \
  --target-key-id <INSIRA_O_KEY_ID> \
  --region us-east-1
```

Em caso de sucesso, o comando não retorna saída.

---

## 3. Replicar para eu-west-1

🔹 **Ambiente:** CloudShell (admin)

A replicação parte sempre da chave primária. O comando referencia a primária (via Key ID) e indica a região de destino.

```bash
aws kms replicate-key \
  --key-id <INSIRA_O_KEY_ID> \
  --replica-region eu-west-1 \
  --description "Réplica para DR em eu-west-1"
```

**Saída (resumida):**

```json
{
    "ReplicaKeyMetadata": {
        "AWSAccountId": "<ACCOUNT_ID>",
        "KeyId": "mrk-1234abcd5678efgh9012ijkl3456mnop",
        "Arn": "arn:aws:kms:eu-west-1:<ACCOUNT_ID>:key/mrk-1234abcd5678efgh9012ijkl3456mnop",
        "Enabled": false,
        "KeyState": "Creating",
        "MultiRegion": true,
        "MultiRegionConfiguration": {
            "MultiRegionKeyType": "REPLICA",
            "PrimaryKey": {
                "Arn": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/mrk-1234abcd5678efgh9012ijkl3456mnop",
                "Region": "us-east-1"
            },
            "ReplicaKeys": [
                {
                    "Arn": "arn:aws:kms:eu-west-1:<ACCOUNT_ID>:key/mrk-1234abcd5678efgh9012ijkl3456mnop",
                    "Region": "eu-west-1"
                }
            ]
        }
    },
    "ReplicaPolicy": "{ ... Key Policy default da réplica ... }"
}
```

> O Key ID da réplica é o mesmo da primária (compartilham o prefixo `mrk-`), mas o ARN é diferente, porque inclui a região `eu-west-1`. O campo `PrimaryKey` aponta de volta para `us-east-1`, e a `ReplicaPolicy` traz a Key Policy própria da réplica (por isso cada região pode ter permissões distintas).

> **Atenção ao estado:** logo após o comando, a réplica aparece com `"KeyState": "Creating"` e `"Enabled": false`. A criação leva alguns segundos. Antes de usar a réplica (passos 5 e 6), confirme que ela está `Enabled`:
>
> ```bash
> aws kms describe-key \
>   --key-id alias/myMultiRegionKey \
>   --query KeyMetadata.KeyState \
>   --output text \
>   --region eu-west-1
> ```

---

## 4. Criar o Alias na Região Réplica

🔹 **Ambiente:** CloudShell (admin), região `eu-west-1`

Diferente do console, o `replicate-key` não cria o alias na réplica. É preciso criá-lo manualmente na segunda região.

```bash
aws kms create-alias \
  --alias-name alias/myMultiRegionKey \
  --target-key-id <INSIRA_O_KEY_ID> \
  --region eu-west-1
```

Em caso de sucesso, o comando não retorna saída.

> Usar o mesmo alias (`alias/myMultiRegionKey`) nas duas regiões simplifica o código: a aplicação referencia o alias e só troca a região. O material é replicado automaticamente, mas o alias e a Key Policy são configurados por região.

---

## 5. Criptografar em us-east-1

🔹 **Ambiente:** CloudShell (admin), região `us-east-1`

Criar um arquivo de teste com uma frase reconhecível:

```bash
echo "Dados sensíveis para teste multi-região" > multiRegionTest.txt
```

Criptografar usando a chave primária, aplicando o Encryption Context:

```bash
aws kms encrypt \
  --key-id alias/myMultiRegionKey \
  --plaintext fileb://multiRegionTest.txt \
  --encryption-context ambiente=dr-test \
  --output text --query CiphertextBlob \
  --region us-east-1 | base64 --decode > multiRegionEncrypted.txt
```

Visualizar o conteúdo cifrado (em base64, para ver os bytes sem bagunçar o terminal):

```bash
base64 multiRegionEncrypted.txt
```

**Saída (trecho):**

```
AQICAHgGwBw7TraMYHUnQoCrHTU26IZh6Tz8bxK/5n8Y/47JFgH/RHQwuRwhTY7w28KBHceoAAAA
iTCBhgYJKoZIhvcNAQcGoHkwdwIBADByBgkqhkiG9w0BBwEwHgYJYIZIAWUDBAEuMBEEDDxutVPN
...
```

O conteúdo original não aparece em lugar nenhum: o arquivo virou uma sequência cifrada ilegível. Sem a chave (e sem o Encryption Context correto), não há como recuperar o texto.

---

## 6. Descriptografar em eu-west-1

🔹 **Ambiente:** CloudShell (admin), região `eu-west-1`

Esta é a demonstração central: ler na segunda região um dado criptografado na primária. Basta trocar o `--region` para `eu-west-1` e informar o mesmo Encryption Context.

```bash
aws kms decrypt \
  --ciphertext-blob fileb://multiRegionEncrypted.txt \
  --encryption-context ambiente=dr-test \
  --output text --query Plaintext \
  --region eu-west-1 | base64 --decode
```

**Saída esperada:**

```
Dados sensíveis para teste multi-região
```

> A descriptografia na segunda região funcionou sem nenhuma etapa intermediária. A réplica usou o mesmo material da primária para ler o dado. Não houve transferência de texto plano entre regiões.

---

## 7. Verificar a Configuração das Chaves

🔹 **Ambiente:** CloudShell (admin)

O `--query KeyMetadata.MultiRegionConfiguration` traz só a parte relevante (tipo da chave e os vínculos entre primária e réplica), sem a saída completa do `describe-key`.

Consultar a chave primária:

```bash
aws kms describe-key \
  --key-id alias/myMultiRegionKey \
  --query "KeyMetadata.MultiRegionConfiguration" \
  --region us-east-1
```

**Saída:**

```json
{
    "MultiRegionKeyType": "PRIMARY",
    "PrimaryKey": {
        "Arn": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/mrk-1234abcd5678efgh9012ijkl3456mnop",
        "Region": "us-east-1"
    },
    "ReplicaKeys": [
        {
            "Arn": "arn:aws:kms:eu-west-1:<ACCOUNT_ID>:key/mrk-1234abcd5678efgh9012ijkl3456mnop",
            "Region": "eu-west-1"
        }
    ]
}
```

Consultar a réplica:

```bash
aws kms describe-key \
  --key-id alias/myMultiRegionKey \
  --query "KeyMetadata.MultiRegionConfiguration" \
  --region eu-west-1
```

**Saída:**

```json
{
    "MultiRegionKeyType": "REPLICA",
    "PrimaryKey": {
        "Arn": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/mrk-1234abcd5678efgh9012ijkl3456mnop",
        "Region": "us-east-1"
    },
    "ReplicaKeys": [
        {
            "Arn": "arn:aws:kms:eu-west-1:<ACCOUNT_ID>:key/mrk-1234abcd5678efgh9012ijkl3456mnop",
            "Region": "eu-west-1"
        }
    ]
}
```

> As duas saídas trazem os mesmos vínculos (`PrimaryKey` em us-east-1 e `ReplicaKeys` em eu-west-1). A diferença está no `MultiRegionKeyType`: `PRIMARY` quando consultada em us-east-1 e `REPLICA` quando consultada em eu-west-1. É a mesma chave lógica, vista de cada região sob o seu papel.

> As validações complementares (fluxo bidirecional e comportamento do Encryption Context entre regiões) estão no doc de [testes](docs/04-testes.md).

---

## Referências

- [Create Multi-Region Primary Key (CLI)](https://docs.aws.amazon.com/cli/latest/reference/kms/create-key.html)
- [Replicate Key (CLI)](https://docs.aws.amazon.com/cli/latest/reference/kms/replicate-key.html)
- [Create Alias (CLI)](https://docs.aws.amazon.com/cli/latest/reference/kms/create-alias.html)
