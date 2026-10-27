# Lições Aprendidas

## O Que Deu Certo

1. **Ajuste da política IAM para menor privilégio**: a política foi restrita a três ações e um ARN específico, limitando o raio de impacto em caso de comprometimento.

2. **Separação entre gerenciamento e uso de chaves**: a Key Policy diferencia quem administra de quem opera. A análise completa está no [doc 08](08-analise-permissoes.md).

3. **Encryption Context como camada adicional**: o par `project=practice` impede a descriptografia sem o valor correto, mesmo com acesso à chave.

4. **Session Manager para acesso seguro**: elimina a necessidade de abrir porta 22, gerenciar pares de chaves SSH ou manter bastion hosts.

---

## O Que Deu Errado e Como Resolvi

### Erro: Esquecer de fornecer o Encryption Context no decrypt

**Sintoma:**
```
An error occurred (InvalidCiphertextException) when calling the Decrypt operation
```

**Causa:** O `--encryption-context` foi usado na criptografia, mas esquecido na descriptografia.

**Solução:**

```bash
aws kms decrypt --encryption-context project=practice --ciphertext-blob fileb://datakeyEncrypted.txt --region us-east-1
```

**Aprendizado:** Se o encryption context foi usado no encrypt, ele é obrigatório no decrypt. Documentar qual valor foi utilizado evita esse tipo de erro.

---

### Erro: Copiar a data key com formatação incorreta

**Sintoma:** Arquivo `datakeyPlainText.txt` gerado com caracteres extras ou quebra de linha.

**Causa:** Ao copiar o valor do Plaintext da saída do terminal, espaços ou caracteres invisíveis foram incluídos.

**Solução:** Copiar o valor para um bloco de notas, verificar que não há espaços antes ou depois, e só então colar no comando:

```bash
echo 'VALOR_EXATO' | base64 --decode > datakeyPlainText.txt
```

**Aprendizado:** Valores base64 são sensíveis a qualquer caractere extra. Usar um bloco de notas intermediário reduz esse risco.

---

### Erro: Tentar criptografar arquivo maior que 4KB diretamente com KMS

**Sintoma:**
```
An error occurred (ValidationException): 1 validation error detected: Value at 'plaintext' failed to satisfy constraint
```

**Causa:** O `aws kms encrypt` só aceita payloads de até 4096 bytes.

**Solução:** Para arquivos maiores, usar Envelope Encryption (gerar data key e criptografar localmente com OpenSSL).

**Aprendizado:** A criptografia direta do KMS é limitada a 4KB. Arquivos maiores exigem Envelope Encryption.


---

## Referências

- [Encryption Context](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#encrypt_context)
- [Limites do KMS (tamanho de payload)](https://docs.aws.amazon.com/kms/latest/developerguide/limits.html)
- [Troubleshooting KMS](https://docs.aws.amazon.com/kms/latest/developerguide/troubleshooting.html)
