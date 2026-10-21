# 📖 Cenário do Laboratório

## Contexto

A rotação de uma chave criptográfica é uma operação sensível. Quando feita sob demanda, fora do ciclo automático, ela precisa ser acompanhada pela equipe de segurança, tanto para confirmar mudanças planejadas quanto para identificar alterações inesperadas. A verificação manual dos logs, porém, não é escalável e depende da inspeção periódica do histórico.

## Problema

- A verificação manual não é escalável e está sujeita a falhas humanas
- Rotações inesperadas podem passar despercebidas sem alertas automáticos
- Não há métricas agregadas sobre a frequência das rotações
- Ausência de notificação automática no momento do evento

Sem monitoramento automático, também fica mais difícil atender a requisitos de auditoria e conformidade, como os da LGPD, HIPAA e PCI-DSS.

## Desafio Proposto

Monitorar a rotação de uma chave KMS e gerar uma notificação automática quando ela ocorre, usando **Amazon EventBridge**, **Amazon SNS** e **Amazon CloudWatch**, sem verificação manual.

A implementação deve permitir:

1. Registrar o evento de rotação com o **AWS CloudTrail**
2. Notificar em tempo real com uma regra do **EventBridge** e um tópico do **SNS**
3. Contabilizar as rotações com um **Metric Filter** e disparar um alarme do **CloudWatch**
4. Comparar as duas abordagens e concluir em que cenário cada uma se encaixa melhor

## Ambiente

| Componente | Detalhe |
|-----------|---------|
| Região | us-east-1 (Virgínia do Norte) |
| Chave monitorada | `rotationMonitorKey` (dedicada a este lab) |
| Fonte dos eventos | AWS CloudTrail (trilha `kms-rotation-trail`) |
| Caminho 1 | Amazon EventBridge + Amazon SNS |
| Caminho 2 | CloudWatch Logs Metric Filter + CloudWatch Alarme |
| Execução | Console da AWS + AWS CloudShell (gatilho da rotação) |

## Pré-requisitos

- Conta AWS com acesso ao console
- Conhecimento básico de KMS e de rotação de chaves (abordada no [Lab 02](../../02-rotacao-de-chaves/))
- Região us-east-1 selecionada no console

---

## Referências

- [Monitoring AWS KMS keys](https://docs.aws.amazon.com/kms/latest/developerguide/monitoring-overview.html)
- [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html)
- [CloudWatch Logs metric filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html)
