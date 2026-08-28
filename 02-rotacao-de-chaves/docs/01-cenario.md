# 📖 Cenário do Laboratório

## Contexto

Após criar e usar uma chave KMS no [Lab de Criptografia](../../01-lab-criptografia/), fica a questão: o que acontece se o material criptográfico dessa chave for comprometido?

Se o mesmo material é usado indefinidamente, o impacto de um vazamento é máximo. Todos os dados criptografados desde a criação da chave estariam vulneráveis. A rotação de chaves existe para limitar esse risco temporal.

## Problema

- Material criptográfico estático ao longo do tempo aumenta a superfície de risco
- Padrões de conformidade (PCI-DSS, SOC 2) exigem rotação periódica de chaves
- Sem rotação, um comprometimento afeta todos os dados históricos protegidos pela chave
- Necessidade de renovar o material sem interromper operações existentes

## Ambiente

| Componente | Detalhe |
|-----------|---------|
| Região | us-east-1 (Virgínia do Norte) |
| Chave KMS | `myKMSKey` (criada no Lab 01) |
| Acesso | AWS Console + AWS CLI |
| Instância | ec2-lab-criptografia (para testes de decrypt) |

## Pré-requisitos

- Chave KMS simétrica existente (criada no Lab 01)
- Permissões `kms:EnableKeyRotation` e `kms:RotateKeyOnDemand` no usuário administrador (a role da EC2 tem apenas permissões de uso: Encrypt, Decrypt e GenerateDataKey)
- Dados já criptografados com a chave (para testar a compatibilidade pós-rotação)
- AWS CLI configurada com acesso à região `us-east-1`

---

## Referências

- [Key Rotation no AWS KMS](https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html)
- [Rotating AWS KMS Keys](https://docs.aws.amazon.com/kms/latest/developerguide/rotating-keys.html)
- [PCI-DSS Key Management Requirements](https://docs.aws.amazon.com/securityhub/latest/userguide/pci-kms-controls.html)
