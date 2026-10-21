# 👤 Configuração IAM

A role criada nesta etapa define o escopo de permissões para administração das chaves, garantindo que o gerenciamento esteja separado das operações criptográficas.

## Passo a Passo via Console

### 1. Acessar o IAM

1. Na barra de navegação superior, digitar: `IAM`
2. Clicar em **IAM** nos resultados

### 2. Criar a Role KMSKeyAdminRole

1. No painel de navegação à esquerda, clicar em **Funções** (Roles)
2. Clicar em **Criar função**
3. Tipo de entidade confiável: **Conta da AWS**
4. Selecionar: **Esta conta** (a própria conta)
5. Clicar em **Avançar**

### 3. Adicionar permissões

As permissões definidas nesta política limitam a atuação da role exclusivamente ao gerenciamento de chaves.

1. Na etapa "Adicionar permissões", selecionar **Criar política em linha**
2. No editor de política, selecionar a aba **JSON**
3. Colar a política abaixo:

```json
{
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
}
```

4. Após colar a política, clicar em **Avançar**



### 4. Nomear, revisar e criar

1. **Nome da função**: `KMSKeyAdminRole`
2. **Descrição**: `Role para administracao de chaves KMS no lab de criptografia`
3. Na seção "Inline policy summary", definir o **Inline policy name**: `KMSKeyAdminPolicy`
4. Revisar a política de confiança e as permissões
5. Clicar em **Criar função**

> Nota: neste lab de teste, operei diretamente com meu usuário IAM de administrador. A `KMSKeyAdminRole` foi criada para demonstrar a separação de responsabilidades (gerenciamento vs uso), padrão recomendado para ambientes com múltiplas pessoas, onde a equipe assumiria a role em vez de ter permissões diretas.

---

## O Que Essa Role Pode e Não Pode Fazer

| Operação | Permitida? |
|----------|:----------:|
| Criar chaves KMS | ✅ |
| Desativar/ativar chaves | ✅ |
| Agendar exclusão de chaves | ✅ |
| Rotacionar chaves sob demanda | ✅ |
| Alterar tags e descrições | ✅ |
| Criptografar dados (Encrypt) | ❌ |
| Descriptografar dados (Decrypt) | ❌ |
| Gerar data keys (GenerateDataKey) | ❌ |

A separação entre gerenciamento e uso permite que diferentes equipes operem com escopos independentes, assumindo a role via SSO/Identity Center com credenciais temporárias.


---

## Referências

- [IAM Roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [IAM Policies e permissões](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)
- [Permissões do KMS via IAM](https://docs.aws.amazon.com/kms/latest/developerguide/control-access-overview.html)
