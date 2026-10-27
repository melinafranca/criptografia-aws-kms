# Arquitetura da Solução

## Diagrama

![Diagrama de Criptografia KMS](assets/diagrama-criptografia-kms.drawio.png)


## Fluxo de Envelope Encryption

![Fluxo de Envelope Encryption - Criptografia](assets/envelope-encryption-fluxo-de-criptografia-kms.drawio.png)

![Fluxo de Envelope Encryption - Descriptografia](assets/envelope-encryption-fluxo-de-descriptografia-kms.drawio.png)

## Fluxo de Criptografia Direta

![Criptografia Direta com KMS - Fluxo de Criptografia](assets/criptografia-direta-com-kms-fluxo-de-criptografia-kms.drawio.png)

![Criptografia Direta com KMS - Fluxo de Descriptografia](assets/criptografia-direta-com-kms-fluxo-de-descriptografia-kms.drawio.png)

## Decisão: Envelope vs Direta

| Critério | Envelope Encryption | Criptografia Direta |
|----------|-------------------|-------------------|
| Tamanho do arquivo | Qualquer tamanho | Até 4KB |
| Performance | Melhor (criptografia local) | Latência de rede |
| Custo | 1 chamada KMS | 1 chamada KMS por operação |
| Complexidade | Maior (gerenciar data key) | Menor |
| Uso típico | Arquivos, bancos de dados | Senhas, tokens, secrets |


---

## Referências

- [Envelope Encryption - Como funciona](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#enveloping)
- [Criptografia direta com KMS (Encrypt API)](https://docs.aws.amazon.com/kms/latest/APIReference/API_Encrypt.html)
- [Decrypt API](https://docs.aws.amazon.com/kms/latest/APIReference/API_Decrypt.html)
- [GenerateDataKey API](https://docs.aws.amazon.com/kms/latest/APIReference/API_GenerateDataKey.html)
