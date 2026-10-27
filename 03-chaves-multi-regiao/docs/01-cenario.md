# Cenário do Laboratório

## Contexto

Organizações que operam em múltiplas regiões AWS precisam garantir acesso aos seus dados criptografados mesmo quando uma região fica indisponível.

O cenário clássico é Disaster Recovery: se a região principal (`us-east-1`) sai do ar, a região secundária (`eu-west-1`) precisa ler os dados criptografados sem depender do KMS da região afetada. Com chaves de região única, isso não é possível, porque o KMS de cada região opera de forma independente e uma chave criada em uma região só descriptografa dados naquela mesma região.

## Problema

- Dados criptografados em uma região não são acessíveis em outra com chaves de região única
- O fluxo tradicional exige descriptografar na origem, que pode estar indisponível
- Transferir dados em texto plano entre regiões é inseguro
- Re-criptografar no destino adiciona complexidade e latência

Em cenários de Disaster Recovery, a dependência da região de origem é justamente o ponto de falha que se quer eliminar. Requisitos de continuidade e conformidade (como PCI-DSS e SOC 2) reforçam a necessidade de acesso confiável aos dados em mais de uma região.

## Desafio Proposto

Implementar chaves multi-região no **AWS Key Management Service (AWS KMS)**.

A implementação deve permitir:

1. Criar uma chave primária multi-região e replicá-la para outra região com o mesmo material criptográfico
2. Criptografar dados em uma região e descriptografá-los em outra diretamente
3. Confirmar que o processo é bidirecional (ambas as regiões criptografam e descriptografam)
4. Validar que o Encryption Context é exigido independente da região de operação

## Ambiente

| Componente | Detalhe |
|-----------|---------|
| Região primária | us-east-1 (Virgínia do Norte) |
| Região réplica | eu-west-1 (Irlanda) |
| Chave KMS | `myMultiRegionKey` (multi-região) |
| Criação e replicação | AWS Console |
| Operações de uso | AWS CloudShell (via CLI) |

## Pré-requisitos

- Conta AWS com acesso ao console e a pelo menos duas regiões
- Conhecimento básico de KMS (chaves, alias e Key Policy)
- Regiões us-east-1 e eu-west-1 disponíveis na conta

---

## Referências

- [Multi-Region Keys in AWS KMS](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-overview.html)
- [Creating Multi-Region Keys](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-create.html)
- [AWS Disaster Recovery Strategies](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html)
