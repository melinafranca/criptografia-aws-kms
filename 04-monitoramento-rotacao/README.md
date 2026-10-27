# Monitoramento de Rotação de Chaves com AWS KMS

![KMS](https://img.shields.io/badge/KMS-Key_Rotation-purple)
![CloudWatch](https://img.shields.io/badge/CloudWatch-Monitoring-blue)
![EventBridge](https://img.shields.io/badge/EventBridge-Events-pink)
![Status](https://img.shields.io/badge/Status-Concluído-green)

## Sobre o projeto

Neste laboratório, aprendi a ser notificada automaticamente quando uma chave KMS é rotacionada, sem depender de verificação manual no CloudTrail.

Este é o quarto passo da jornada com o KMS: depois de proteger dados no [Lab 01](../01-lab-criptografia/), rotacionar chaves no [Lab 02](../02-rotacao-de-chaves/) e replicar entre regiões no [Lab 03](../03-chaves-multi-regiao/), o foco agora é a observabilidade da rotação.

O lab implementa e compara dois caminhos de monitoramento. Toda a configuração foi feita no console da AWS, e o gatilho da rotação, via AWS CloudShell.

---

## O desafio

O [Lab 02](../02-rotacao-de-chaves/) validou a rotação olhando o CloudTrail manualmente. Isso não escala: ninguém acompanha o histórico todo dia. Uma rotação, principalmente uma sob demanda feita fora do ciclo, é um evento que a equipe de segurança precisa saber que aconteceu, na hora.

O desafio foi transformar esse evento em uma notificação automática e comparar duas formas de fazer isso na AWS.

---

## Objetivo

Monitorar a rotação da chave `rotationMonitorKey` por dois caminhos independentes e comparar as abordagens:

1. **EventBridge + SNS** — captura o evento e envia um email, quase em tempo real
2. **CloudWatch Metric Filter + Alarme** — conta as rotações e dispara um alarme por threshold

---

## O que foi implementado

* Criação de uma chave dedicada (`rotationMonitorKey`) com rotação automática ativada
* Criação de uma trilha do CloudTrail entregando eventos para o CloudWatch Logs
* **Caminho 1:** regra do EventBridge filtrando `RotateKeyOnDemand` + tópico SNS com assinatura por email
* **Caminho 2:** metric filter contando as rotações + alarme do CloudWatch com notificação por SNS
* Validação dos dois caminhos disparando uma rotação sob demanda

---

## Tecnologias e serviços utilizados

| Serviço / Ferramenta | Função no projeto |
|----------------------|-------------------|
| AWS KMS | Chave monitorada e rotação sob demanda |
| AWS CloudTrail | Registro do evento de rotação |
| Amazon EventBridge | Captura do evento e encaminhamento (Caminho 1) |
| Amazon SNS | Entrega das notificações por email |
| Amazon CloudWatch | Metric filter e alarme (Caminho 2) |
| AWS CloudShell | Gatilho da rotação via CLI |

---

## Arquitetura

Os dois caminhos partem do mesmo evento do CloudTrail e chegam ao mesmo destino (email), por rotas diferentes:

```
Caminho 1: Rotação → CloudTrail → EventBridge → SNS → Email
Caminho 2: Rotação → CloudTrail → Log Group → Metric Filter → Alarme → SNS → Email
```

A comparação detalhada das duas abordagens (latência, custo, quando usar cada uma) está no doc de [arquitetura](docs/02-arquitetura.md).

---

## Documentação

| # | Documento | Descrição |
|---|-----------|-----------|
| 1 | [Cenário do laboratório](docs/01-cenario.md) | Contexto e problema abordado |
| 2 | [Arquitetura da solução](docs/02-arquitetura.md) | Os dois caminhos e a comparação |
| 3 | [Passo a passo](docs/03-passo-a-passo.md) | Configuração da chave, trilha e dos dois caminhos |
| 4 | [Testes e validação](docs/04-testes.md) | Validação dos dois caminhos |
| 5 | [Lições aprendidas](docs/05-licoes-aprendidas.md) | Descobertas e troubleshooting |

---

## Investimento

| Recurso | Custo estimado |
|---------|---------------|
| Chave KMS (armazenamento mensal) | ~$1.00/mês |
| CloudTrail (eventos de gerenciamento) | Primeira trilha gratuita |
| S3 (armazenamento dos logs) | Centavos, conforme volume |
| CloudWatch Logs (ingestão) | Centavos, conforme volume |
| SNS (email) | Dentro do nível gratuito |
| **Total estimado** | **~$1.00/mês + centavos de logs** |

> A primeira trilha de eventos de gerenciamento não tem custo adicional de CloudTrail. O custo maior vem do armazenamento contínuo dos logs (S3 e CloudWatch), por isso vale definir retenção ou remover os recursos ao final.

---

## Resultados

Ao final do laboratório, foi possível:

* Configurar uma chave monitorada e uma trilha do CloudTrail
* Notificar a rotação por email através do EventBridge (Caminho 1)
* Contar as rotações e disparar um alarme através do CloudWatch (Caminho 2)
* Comparar as duas abordagens e medir a diferença de latência entre elas
* Confirmar, na prática, que os dois caminhos são complementares

---

## Autora

**Melina França** —  AWS Developer Associate

---

⬅️ [Voltar ao repositório principal](../README.md)
