# Testes e Validação

Este documento reúne os testes que provam comportamentos além do fluxo básico. A criação, a replicação e o ciclo simples de criptografia (encrypt em `us-east-1`, decrypt em `eu-west-1`) já foram validados no [guia via CLI](../lab-via-cli.md) e não são repetidos aqui.

Os testes foram executados no **AWS CloudShell**, alternando a região pelo parâmetro `--region`.

---

## Teste 1: Fluxo Bidirecional

**Objetivo:** Confirmar que a réplica não é só um destino de leitura. Ela também criptografa, e o dado gerado nela é lido pela primária.

Criar um arquivo e criptografá-lo na **réplica** (`eu-west-1`):

```bash
echo "Criptografado na replica" > replicaTest.txt
```

```bash
aws kms encrypt \
  --key-id alias/myMultiRegionKey \
  --plaintext fileb://replicaTest.txt \
  --encryption-context ambiente=dr-test \
  --output text --query CiphertextBlob \
  --region eu-west-1 | base64 --decode > replicaEncrypted.txt
```

Descriptografar na **primária** (`us-east-1`):

```bash
aws kms decrypt \
  --ciphertext-blob fileb://replicaEncrypted.txt \
  --encryption-context ambiente=dr-test \
  --output text --query Plaintext \
  --region us-east-1 | base64 --decode
```

**Saída:**

```
Criptografado na replica
```

**Resultado:** ✅ O sentido inverso funciona. A réplica criptografou e a primária leu o dado. As duas chaves são funcionalmente equivalentes: não há uma relação fixa de "origem e destino", qualquer uma das regiões criptografa e descriptografa.

---

## Teste 2: Encryption Context entre Regiões

**Objetivo:** Confirmar que o Encryption Context é exigido na descriptografia mesmo quando ela acontece em outra região. O arquivo `multiRegionEncrypted.txt` foi criptografado em `us-east-1` com o context `ambiente=dr-test`.

### 2a. Sem Encryption Context

```bash
aws kms decrypt \
  --ciphertext-blob fileb://multiRegionEncrypted.txt \
  --output text --query Plaintext \
  --region eu-west-1
```

**Saída:**

```
An error occurred (InvalidCiphertextException) when calling the Decrypt operation
```

### 2b. Com Encryption Context errado

```bash
aws kms decrypt \
  --ciphertext-blob fileb://multiRegionEncrypted.txt \
  --encryption-context ambiente=errado \
  --output text --query Plaintext \
  --region eu-west-1
```

**Saída:**

```
An error occurred (InvalidCiphertextException) when calling the Decrypt operation
```

**Resultado:** ✅ Os dois casos falharam com `InvalidCiphertextException`, mesmo a operação rodando em `eu-west-1`, longe da região onde o dado foi criptografado. O Encryption Context é validado independente da região.

> O erro é o mesmo ao omitir o context (2a) e ao usar um errado (2b). O KMS não distingue "faltou context" de "context incorreto": em ambos os casos, ele não consegue validar o ciphertext e recusa a operação. O Encryption Context faz parte do que torna o ciphertext decifrável, não é um parâmetro opcional.

---

## Conclusão

A funcionalidade multi-região é transparente e bidirecional: qualquer uma das regiões criptografa e descriptografa, sem transferência de texto plano. O Encryption Context é respeitado em qualquer região, mantendo a mesma camada de segurança de ponta a ponta. Aplicações existentes não precisam de alteração no código além da região informada na chamada.
