# 🏗️ Arquitetura da Solução

## O Que É?

A rotação de chaves no KMS gera um novo material criptográfico dentro de uma chave que já existe. Não cria uma chave nova: o Key ID, o ARN e o alias permanecem os mesmos. O KMS troca o segredo interno, mas preserva a identidade da chave, e guarda todas as versões anteriores do material para manter os dados antigos decifráveis.

## Por Que Usar?

- Limitar o escopo temporal de um possível comprometimento
- Atender requisitos de conformidade (PCI-DSS, SOC 2) sem intervenção manual
- Garantir que novas operações usem material atualizado automaticamente
- Manter a operação transparente para aplicações existentes

---

## Fluxo da Operação

![Arquitetura da rotação de chaves KMS](assets/arquitetura-rotacao.png)

A instância EC2 se comunica com o KMS para duas operações: criptografar novos dados (Encrypt) e descriptografar dados existentes (Decrypt). O KMS decide internamente qual versão do material usar: a mais recente (V2) para os dados novos e a anterior (V1) para os dados antigos. O texto legível é devolvido à EC2 sem exigir nenhuma alteração de código ou de chave.

No console, essas versões aparecem na tabela **Materiais de chaves**: o V2 do diagrama corresponde ao material marcado como **Atual**, e o V1 ao material **Não atual**. Foi essa correspondência que os testes confirmaram, comparando o `KeyMaterialId` de cada operação.

---

## Fluxo Temporal

A cada rotação, o KMS adiciona uma nova versão de material e passa a usá-la nas novas criptografias. As versões anteriores continuam disponíveis para descriptografar o que já foi criptografado com elas.

```
Ano 0: Chave criada com Material v1
        Encrypt → usa v1
        Decrypt → usa v1

Ano 1: Rotação → Material v2 gerado
        Encrypt → usa v2 (mais recente)
        Decrypt → KMS escolhe v1 ou v2 conforme o dado

Ano 2: Rotação → Material v3 gerado
        Encrypt → usa v3
        Decrypt → KMS escolhe a versão correta automaticamente
```

---

## O Que NÃO Muda Após Rotação

| Elemento | Muda? | Detalhe |
|----------|:-----:|---------|
| Key ID | ❌ | Permanece o mesmo |
| Key ARN | ❌ | Permanece o mesmo |
| Alias | ❌ | Continua apontando para a mesma chave |
| Key Policy | ❌ | Permissões não são afetadas |
| Código da aplicação | ❌ | Nenhuma alteração necessária |
| Dados antigos | ❌ | Continuam descriptografáveis |

---

## Quando Usar Cada Tipo de Rotação

```
Preciso rotacionar a chave?
│
├── Operação padrão de segurança
│   └── Rotação automática (anual)
│       └── Sem custo extra
│
├── Suspeita de comprometimento
│   └── Rotação sob demanda (imediata)
│       └── $1.00 por rotação
│
└── Comprometimento do acesso à chave (role/policy)
    └── Desativar a chave + criar uma nova
        └── Re-criptografar os dados
```

> A rotação protege contra o comprometimento do material criptográfico. Se o acesso à chave em si for comprometido (não apenas o material), rotação não resolve: é necessário desativar a chave e re-criptografar os dados com uma nova.

---

## Referências

- [How Key Rotation Works](https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html#rotate-keys-how-it-works)
- [On-demand Key Rotation](https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys-on-demand.html)
