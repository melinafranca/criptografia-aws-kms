# 🏗️ Arquitetura da Solução

## O Que É?

Chaves multi-região são chaves KMS em diferentes regiões AWS que compartilham o mesmo material criptográfico. Diferente de criar chaves separadas em cada região, elas permitem que um ciphertext gerado em uma região seja descriptografado em outra diretamente.

## Por Que Usar?

- Permitir Disaster Recovery sem dependência da região de origem
- Eliminar a necessidade de transferir dados em texto plano entre regiões
- Reduzir latência de decrypt para usuários em outra região
- Manter o código da aplicação simples (mesmo alias, só muda a região)

---

## Fluxo da Operação

![Arquitetura das operações de criptografia com chaves multi-região](assets/aws-kms-operacoes-de-criptografia-com-chaves-multi-regiao.png)

A chave primária é criada no console em `us-east-1` e replicada para `eu-west-1`. As duas chaves compartilham o mesmo material criptográfico. Pelo CloudShell, a criptografia acontece na primária (us-east-1) e a descriptografia na réplica (eu-west-1), cada operação chamando o KMS da sua região.

> O KMS garante que o ciphertext gerado em uma região possa ser lido na outra. A transferência do arquivo criptografado entre regiões é responsabilidade da aplicação (por exemplo, via S3 Cross-Region Replication). Multi-região resolve o acesso à chave, não o transporte dos dados.

---

## Diferença: Região Única vs Multi-Região

| Aspecto | Região Única | Multi-Região |
|---------|:------------:|:------------:|
| Encrypt em outra região | ❌ Precisa usar KMS da origem | ✅ Réplica funciona independente |
| Decrypt em outra região | ❌ Precisa usar KMS da origem | ✅ Réplica funciona independente |
| Disaster Recovery | Depende da região estar disponível | Independente entre regiões |
| Key ID | `xxxxxxxx-xxxx-xxxx-xxxx` | `mrk-xxxxxxxx` (prefixo mrk) |
| Custo | $1/mês (1 chave) | $1/mês por região |

---

## Relação entre Primária e Réplica

A chave primária é a "dona" do material criptográfico. A réplica é uma cópia funcional independente.

| Operação | Primária | Réplica |
|----------|:--------:|:-------:|
| Encrypt | ✅ | ✅ |
| Decrypt | ✅ | ✅ |
| Gerenciar (policy, tags, alias) | ✅ | ✅ (própria por região) |
| Replicar para outra região | ✅ | ❌ (só a primária replica) |
| Deletar | ✅ (afeta réplicas) | ✅ (só a réplica é removida) |

> A Key Policy, as tags e o alias de cada região são independentes. Isso permite que equipes de segurança em regiões diferentes tenham configurações distintas, respeitando a governança local.

---

## Referências

- [Multi-Region Keys Overview](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-overview.html)
- [How Multi-Region Keys Work](https://docs.aws.amazon.com/kms/latest/developerguide/mrk-how-it-works.html)
