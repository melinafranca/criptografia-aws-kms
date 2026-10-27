# Análise de Permissões

Este documento registra as decisões de segurança e o raciocínio por trás de cada permissão configurada.

---

## Key Policy: O Que Cada Statement Faz

A Key Policy gerada na criação da chave `myKMSKey` possui 4 statements. Cada um foi criado automaticamente com base nas escolhas feitas no wizard:

| Statement | Gerado por | Função |
|-----------|-----------|--------|
| Enable IAM User Permissions | Padrão do KMS | Permite que a conta root gerencie a chave via IAM policies |
| Allow access for Key Administrators | Etapa 3 (admin) | Permite gerenciar a chave (sem usar para criptografia) |
| Allow use of the key | Etapa 4 (user) | Permite operações criptográficas |
| Allow attachment of persistent resources | Gerado com Etapa 4 | Permite que serviços AWS usem grants para acessar a chave |

O primeiro statement (`kms:*` para root) é intencional: garante que a conta sempre tenha controle sobre a chave. Se todos os administradores forem removidos acidentalmente, a conta root ainda pode recuperar o acesso. Sem esse statement, é possível criar uma chave irrecuperável.

---

## Separação de Responsabilidades: Admin vs User

A separação entre gerenciamento e uso da chave foi uma decisão central neste projeto. O lab foi operado inteiramente com um usuário IAM com permissões de administrador, sem uso do usuário root.

| Papel | O que pode fazer | O que não pode fazer | Observação |
|-------|-----------------|---------------------|------------|
| Root (Key Policy) | Tudo sobre a chave | — | Statement padrão do KMS para recuperação de acesso. Não foi utilizado neste lab. |
| Key Administrator (usuário IAM) | Criar, desativar, deletar, rotacionar chaves | Criptografar ou descriptografar dados | Operador do lab com permissões de administrador |
| EC2 Role (`EC2KMSLabRole`) | Criptografar, descriptografar, gerar data keys | Deletar a chave, alterar políticas, desativar rotação | Permissões mínimas para o lab funcionar |
| Role administrativa (`KMSKeyAdminRole`) | Gerenciar chaves (pronta para uso em equipe) | Criptografar ou descriptografar dados | Criada como boa prática para cenários colaborativos |

Em caso de comprometimento da instância, o impacto fica contido às operações permitidas pela role (criptografar e descriptografar). A exclusão da chave, alteração de políticas e escalonamento de privilégios permanecem inacessíveis.

---

## Política Permissiva vs Restritiva

### Política padrão em ambientes de teste:

```json
{
  "Action": "kms:*",
  "Resource": "*",
  "Effect": "Allow"
}
```

Isso concede 40+ ações do KMS em qualquer chave da conta. Funciona, mas abre espaço para:
- Deletar chaves acidentalmente
- Acessar chaves de outros projetos/equipes
- Alterar políticas sem supervisão

### O que implementamos neste lab:

```json
{
  "Action": [
    "kms:GenerateDataKey",
    "kms:Encrypt",
    "kms:Decrypt"
  ],
  "Resource": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/<KEY_ID>"
}
```

Três ações. Um recurso específico. O mínimo necessário para o lab funcionar.

### Comparação visual:

| Aspecto | Política permissiva | Política restritiva (implementada) |
|---------|:------------------:|:---------------------------------:|
| Ações concedidas | 40+ | 3 |
| Recursos afetados | Todas as chaves da conta | Uma chave específica |
| Pode deletar chaves | ✅ | ❌ |
| Pode alterar políticas | ✅ | ❌ |
| Pode desativar rotação | ✅ | ❌ |
| Pode escalar privilégios | ✅ | ❌ |
| Funciona para o lab | ✅ | ✅ |

Ambas fazem o lab funcionar. A diferença é o raio de impacto em caso de problema.

---

## Decisão de Design

A política original do CloudQuest utiliza `kms:*` para simplificar o ambiente. Ao reproduzir no console, optei por restringir as permissões ao escopo necessário:

1. **Raio de impacto limitado**: em caso de erro ou comprometimento, a instância opera apenas com uma chave específica e três ações definidas.

2. **Reprodutibilidade segura**: quem seguir este laboratório não cria acidentalmente um ambiente com permissões excessivas.

3. **Segurança como parte da solução**: restringir permissões demonstra que a solução foi pensada não apenas para funcionar, mas para funcionar de forma segura.

---

## Resumo das Permissões do Lab

| Recurso | Quem | Permissões | Princípio aplicado |
|---------|------|-----------|-------------------|
| Chave KMS | Key Admin (usuário IAM) | Gerenciar (sem criptografar) | Separação de responsabilidades |
| Chave KMS | EC2 Role | Criptografar/descriptografar (sem gerenciar) | Menor privilégio |
| Instância EC2 | Session Manager | Comunicação com SSM | Acesso sem SSH |
| Security Group | Nenhuma entrada | Zero portas abertas | Superfície de ataque mínima |


---

## Auditoria das Operações

O AWS CloudTrail registra automaticamente todas as chamadas de API ao KMS. No console, é possível consultar via CloudTrail → Event history, filtrando por `kms.amazonaws.com`. Cada evento registra quem executou, qual operação, qual chave e de onde veio a chamada.

---

## Integração do KMS com Outros Serviços

O KMS se integra nativamente com diversos serviços AWS:

- **Amazon S3**: SSE-KMS para criptografia de objetos armazenados
- **Amazon EBS**: criptografia de volumes anexados a instâncias EC2
- **Amazon RDS**: criptografia de bancos de dados em repouso
- **AWS Secrets Manager**: armazenamento e rotação de segredos
- **AWS CloudTrail**: auditoria de todas as operações realizadas com as chaves

Essas integrações utilizam o mesmo modelo de Envelope Encryption explorado neste laboratório. O serviço gera uma data key via KMS, criptografa os dados localmente e armazena a cópia criptografada da chave junto com os dados protegidos.


---

## Referências

- [Key Policies](https://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html)
- [Princípio do menor privilégio](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege)
- [AWS CloudTrail - Registrar eventos do KMS](https://docs.aws.amazon.com/kms/latest/developerguide/logging-using-cloudtrail.html)
- [Serviços integrados ao KMS](https://docs.aws.amazon.com/kms/latest/developerguide/service-integration.html)
