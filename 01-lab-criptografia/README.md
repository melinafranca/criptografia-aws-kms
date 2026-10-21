# 🔐 Lab de Criptografia com AWS KMS

![EC2](https://img.shields.io/badge/EC2-Instance-blue)
![KMS](https://img.shields.io/badge/KMS-Encryption-purple)
![IAM](https://img.shields.io/badge/IAM-Security-red)

## 📋 Sobre o projeto

Neste laboratório, explorei diferentes formas de proteger dados armazenados em uma instância Amazon EC2 utilizando o **AWS Key Management Service (AWS KMS)**.

O foco foi implementar dois modelos de criptografia:

* **Envelope Encryption**, utilizando uma data key gerada pelo AWS KMS para criptografar arquivos com OpenSSL;
* **Criptografia direta com AWS KMS**, voltada para arquivos pequenos.

A prática foi inspirada no **AWS Cloud Quest: Serverless Developer** e na **Trilha de Extensão Developer da Escola da Nuvem**. Após concluir o cenário no sandbox do CloudQuest, reproduzi a implementação em um **ambiente de desenvolvimento na AWS**, utilizando o console e a AWS CLI para validar o funcionamento da solução e documentar os resultados.

---

## 🎯 O desafio

Dados armazenados em servidores podem conter informações sensíveis, como senhas, tokens ou dados pessoais. Quando esses arquivos permanecem em texto aberto, qualquer acesso indevido ao ambiente pode representar um risco à confidencialidade dessas informações.

Neste projeto, o desafio foi aplicar o AWS KMS à proteção desses dados e avaliar como diferentes abordagens de criptografia se adequam a diferentes cenários.

Além da implementação, também explorei aspectos relacionados ao controle de acesso, ao princípio do menor privilégio e à recuperação segura dos dados após a criptografia.

---

## 🏆 Objetivo

Implementar e validar dois métodos de criptografia utilizando o AWS KMS:

### 1. Envelope Encryption

Gerar uma **data key** utilizando o AWS KMS e utilizá-la para criptografar arquivos localmente com o OpenSSL.

Essa abordagem permite trabalhar com arquivos maiores, enquanto o AWS KMS é utilizado para proteger a chave responsável pela criptografia dos dados.

### 2. Criptografia direta com AWS KMS

Utilizar diretamente as operações de criptografia e descriptografia do AWS KMS para proteger arquivos pequenos.

Nesse caso, os dados são enviados ao KMS para serem criptografados, respeitando o limite de tamanho suportado pela operação.

---

## 🛠️ O que foi implementado

Durante a implementação, foram realizadas as seguintes etapas:

* Criação de uma chave simétrica no AWS KMS;
* Configuração de permissões de acesso utilizando IAM e Key Policies;
* Criação e configuração de uma instância Amazon EC2;
* Acesso seguro à instância utilizando o AWS Systems Manager Session Manager;
* Geração de uma data key para implementação do Envelope Encryption;
* Criptografia e descriptografia de arquivos utilizando OpenSSL;
* Exclusão da chave em texto aberto após sua utilização;
* Recuperação da data key utilizando o AWS KMS;
* Criptografia direta de arquivos pequenos utilizando as operações `Encrypt` e `Decrypt`;
* Testes de permissões e validação do princípio do menor privilégio;
* Utilização do Encryption Context como camada adicional de segurança.

---

## 🧰 Tecnologias e serviços utilizados

| Serviço / Ferramenta | Função no projeto |
|----------------------|-------------------|
| Amazon EC2 | Ambiente Linux utilizado para executar os testes e comandos |
| AWS KMS | Criação e gerenciamento das chaves criptográficas |
| AWS IAM | Controle de acesso aos recursos e operações do KMS |
| AWS Systems Manager | Acesso seguro à instância por meio do Session Manager |
| OpenSSL | Criptografia e descriptografia dos arquivos utilizando a data key |
| AWS CLI | Execução das operações do AWS KMS pela linha de comando |

---

## 🏗️ Arquitetura

![Arquitetura da Solução](docs/assets/diagrama-criptografia-kms.drawio.png)

A arquitetura utiliza uma instância Amazon EC2 como ambiente de execução. O acesso às funcionalidades de criptografia é controlado por permissões do IAM e pelas políticas associadas à chave do AWS KMS.

No fluxo de **Envelope Encryption**, o KMS é responsável pela geração e proteção da data key, enquanto a criptografia do arquivo é realizada localmente com o OpenSSL. **Na criptografia direta, os dados são enviados ao KMS para execução das operações de criptografia e descriptografia, respeitando o limite de tamanho suportado por esse método.**

---

## 📖 Documentação

A documentação foi dividida em etapas para facilitar o acompanhamento da implementação:

| # | Documento | Descrição |
|---|-----------|-----------|
| 1 | 📖 [Cenário do laboratório](docs/01-cenario.md) | Contexto e problema abordado |
| 2 | 🏗️ [Arquitetura da solução](docs/02-arquitetura.md) | Componentes e fluxo da solução |
| 3 | 👤 [Configuração IAM](docs/03-configuracao-iam.md) | Criação e configuração das permissões |
| 4 | 🔑 [Criação da chave KMS](docs/04-criacao-chave-kms.md) | Configuração da chave utilizada no laboratório |
| 5 | 🖥️ [Criação da EC2](docs/05-criacao-ec2.md) | Configuração da instância e do acesso via Session Manager |
| 6 | 📦 [Envelope Encryption](docs/06-envelope-encryption.md) | Geração da data key e criptografia com OpenSSL |
| 7 | 🔐 [Criptografia direta com KMS](docs/07-criptografia-direta.md) | Utilização das operações Encrypt e Decrypt |
| 8 | 🔒 [Análise de permissões](docs/08-analise-permissoes.md) | Controle de acesso, auditoria e integrações |
| 9 | 💡 [Lições aprendidas](docs/09-licoes-aprendidas.md) | Troubleshooting e principais aprendizados |

> Este laboratório também pode ser reproduzido inteiramente via AWS CLI. Consulte o [guia via CLI](lab-via-cli.md).

> Os recursos criados neste laboratório também serão utilizados em projetos posteriores relacionados à rotação de chaves e à utilização de chaves Multi-Region. A remoção dos recursos está documentada no [guia de remoção](../REMOCAO-RECURSOS.md) do repositório principal.

---

## 💰 Custos

| Recurso | Custo estimado |
|---------|---------------|
| Amazon EC2 | Utilização dentro das condições do Free Tier |
| AWS KMS | ~$1.00/mês por chave armazenada |
| Requisições ao KMS | Cobrança conforme a quantidade de operações realizadas |
| AWS Systems Manager Session Manager | Sem custo adicional para o acesso à instância |
| **Custo observado** | **Inferior a US$ 1,00 durante os testes realizados** |

> O laboratório foi inicialmente desenvolvido em um ambiente sandbox e posteriormente reproduzido em um **ambiente de desenvolvimento na AWS**. Os recursos foram utilizados apenas durante os testes e dentro de um período limitado.

---

## ✅ Resultados

Ao final do laboratório, foi possível:

* Criar e configurar uma chave simétrica no AWS KMS;
* Implementar o fluxo completo de Envelope Encryption;
* Gerar, utilizar e recuperar uma data key;
* Criptografar e descriptografar arquivos utilizando OpenSSL;
* Validar a exclusão da chave em texto aberto após seu uso;
* Implementar criptografia direta com o AWS KMS para dados dentro do limite suportado pela operação;
* Testar o Encryption Context como uma camada adicional de proteção;
* Validar o controle de acesso por meio de IAM e Key Policies;
* Recuperar com sucesso os dados originais após o processo de descriptografia.

Na prática, o projeto permitiu comparar dois modelos de criptografia e compreender quando o AWS KMS pode atuar diretamente sobre os dados e quando é mais adequado utilizá-lo para proteger as chaves responsáveis pela criptografia.

---

## 👩‍💻 Autora

**Melina França** —  AWS Developer Associate

---

⬅️ [Voltar ao repositório principal](../README.md)
