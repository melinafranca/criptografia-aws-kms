# 📖 Cenário do Laboratório

## Contexto

Dados armazenados em ambientes de nuvem frequentemente contêm informações que exigem proteção contra acessos não autorizados.

Em instâncias EC2, arquivos com credenciais, tokens ou dados pessoais podem permanecer em texto plano quando não são protegidos por mecanismos de criptografia. Qualquer pessoa com acesso à instância pode ler esses arquivos diretamente.

## Problema

- Dados sensíveis armazenados sem criptografia
- Possibilidade de acesso não autorizado às informações
- Ausência de uma camada de proteção para dados em repouso
- Necessidade de controlar quem pode realizar operações de criptografia e descriptografia

A ausência de controles adequados de proteção de dados também pode dificultar o atendimento a requisitos de segurança e privacidade, como os previstos na LGPD, HIPAA e PCI-DSS, conforme o contexto da organização.

## Desafio Proposto

Implementar uma solução de criptografia para dados em repouso utilizando o **AWS Key Management Service (AWS KMS)**.

A implementação deve permitir:

1. Criar e gerenciar a chave utilizada no processo
2. Utilizar **Envelope Encryption** para proteger arquivos de qualquer tamanho
3. Utilizar **criptografia direta via KMS** para dados de até 4KB (limite da operação direta do KMS)
4. Controlar o acesso às operações criptográficas por meio de IAM e Key Policies

## Ambiente

| Componente | Detalhe |
|-----------|---------|
| Região | us-east-1 (Virgínia do Norte) |
| Instância | ec2-lab-criptografia (EC2 Linux) |
| Acesso | AWS Systems Manager (Session Manager) |
| Usuário | ec2-user |

## Pré-requisitos

- Conta AWS com acesso ao console
- Conhecimento básico de IAM (roles e policies)
- Região us-east-1 selecionada no console


---

## Referências

- [AWS Key Management Service (KMS)](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html)
- [Envelope Encryption](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#enveloping)
- [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
