# 🔐 Criptografia com AWS KMS

![AWS](https://img.shields.io/badge/AWS-Cloud-orange?style=flat&logo=amazon-aws)
![Status](https://img.shields.io/badge/Status-Concluído-green)
![KMS](https://img.shields.io/badge/KMS-Encryption-purple)

## 📋 Sobre o Repositório

Este repositório reúne laboratórios sobre criptografia com AWS KMS. O projeto foi inspirado no AWS Cloud Quest Serverless Developer e na Trilha Developer da Escola da Nuvem. Durante a implementação, reconstruí o ambiente, documentei as decisões de segurança e venho expandindo o escopo para incluir rotação de chaves, monitoramento e replicação entre regiões.

## 🗺️ Roadmap da Jornada

| # | Projeto | Descrição | Status |
|---|---------|-----------|--------|
| 1 | [Lab de Criptografia](01-lab-criptografia/) | Envelope Encryption e criptografia direta via CLI | ✅ Concluído |
| 2 | [Rotação de Chaves](02-rotacao-de-chaves/) | Rotação automática e sob demanda de material criptográfico | ✅ Concluído |
| 3 | [Chaves Multi-Região](03-chaves-multi-regiao/) | Replicação de chaves entre regiões para Disaster Recovery | ✅ Concluído |
| 4 | [Monitoramento de Rotação](04-monitoramento-rotacao/) | Notificação automática de rotação via EventBridge/SNS e CloudWatch | ✅ Concluído |

## 🧰 Serviços Utilizados na Jornada

| Serviço | Função |
|---------|--------|
| AWS KMS | Criação e gerenciamento de chaves criptográficas |
| Amazon EC2 | Instância Linux para execução dos comandos |
| AWS IAM | Controle de acesso via roles e policies |
| AWS Systems Manager | Acesso seguro à instância via Session Manager |
| AWS CloudShell | Execução de comandos da AWS CLI, incluindo operações entre regiões |
| AWS CloudTrail | Auditoria de uso das chaves |
| OpenSSL | Criptografia local com data keys |

## 💡 Sobre a Jornada

Este repositório demonstra:

- Proteção de dados em repouso com diferentes abordagens
- Controle de acesso granular às chaves criptográficas
- Planejamento de continuidade (rotação e multi-região)
- Capacidade de diagnosticar erros e entender o comportamento do serviço

## 🗑️ Remoção de Recursos

Ao concluir os laboratórios, consulte o [guia de remoção de recursos](REMOCAO-RECURSOS.md) para excluir todos os recursos criados de forma segura e na ordem correta.

## 👩‍💻 Autora

**Melina França** —  AWS Developer Associate

---

⭐ Se este repositório te ajudou, deixe uma estrela!
