# Lições Aprendidas

## O Que Deu Certo

1. **Os dois caminhos funcionaram de ponta a ponta**: uma rotação sob demanda gerou notificação tanto pelo EventBridge (Caminho 1) quanto pelo alarme do CloudWatch (Caminho 2).

2. **Filtro específico elimina o ruído**: filtrar pelo `eventName` `RotateKeyOnDemand` fez os alertas dispararem só na rotação, ignorando a enxurrada de outros eventos que o CloudTrail registra.

3. **CloudShell como gatilho**: rodar a rotação sob demanda pelo CloudShell foi suficiente para disparar toda a cadeia, sem infraestrutura adicional.

4. **Diferença de latência confirmada**: o EventBridge notificou quase em tempo real; o CloudWatch levou ~2,5 minutos. As duas abordagens se provaram complementares.

---

## O Que Deu Errado e Como Resolvi

### O email não chegou porque faltava a trilha do CloudTrail

**Sintoma:** com tudo configurado (regra do EventBridge, filtro, tópico SNS e assinatura confirmada), a rotação sob demanda não gerava nenhum email. O evento aparecia normalmente no Event history do CloudTrail.

**Causa:** o Event history mostra os eventos, mas não os entrega para outros serviços. Sem uma trilha ativa, os eventos de gerenciamento não chegam ao EventBridge, então a regra nunca dispara.

**Solução:** criar a trilha `kms-rotation-trail`. Com ela ativa, a rotação seguinte disparou a cadeia completa e o email chegou.

**Aprendizado:** o ponto de falha era invisível, todos os recursos estavam corretos, e mesmo assim nada funcionava, porque faltava a fundação. Ver o evento no Event history não garante que ele será entregue. Confirmar a trilha antes de depurar a regra.

---

### A armadilha "Excluir eventos do AWS KMS"

**Sintoma:** risco de a trilha não registrar justamente o evento que o lab precisa.

**Causa:** na criação da trilha, a opção **Excluir eventos do AWS KMS** (Atividade da API) faria a trilha ignorar os eventos do KMS, incluindo o `RotateKeyOnDemand`.

**Solução:** deixar essa opção **desmarcada**.

**Aprendizado:** é uma armadilha sutil, a opção existe exatamente para o serviço que estamos monitorando. Se marcada por engano, o fluxo falharia silenciosamente: a rotação aconteceria, mas nunca seria logada nem notificada.

---

### Dois "SNS" diferentes, fácil de confundir

**Sintoma:** confusão ao configurar a trilha, que também oferece uma opção de SNS.

**Causa:** existem dois usos de SNS no lab. O **SNS da trilha** ("Entrega de notificações do SNS") avisa sobre entrega de arquivos de log no S3. O **SNS do lab** (`kms-rotation-alerts`) é o que entrega o alerta de rotação.

**Solução:** deixar a opção de SNS da trilha desmarcada e usar o tópico do lab apenas nos destinos do EventBridge e do alarme.

**Aprendizado:** o mesmo nome de serviço aparecendo em contextos diferentes é uma fonte de erro. Vale confirmar qual SNS está sendo configurado antes de marcar.

---

### O teste de padrão do console avalia um stream por vez

**Sintoma:** o metric filter, testado no console, não retornava nenhuma correspondência, mesmo com o padrão correto.

**Causa:** o botão "Testar padrão" avalia um único log stream selecionado. O evento de rotação estava em outro stream.

**Solução:** validar o padrão contra o Log Group inteiro via CLI:

```bash
aws logs filter-log-events \
  --log-group-name aws-cloudtrail-logs-kms-rotation \
  --filter-pattern '{ $.eventName = "RotateKeyOnDemand" }' \
  --region us-east-1
```

**Aprendizado:** o teste visual do console é limitado ao stream escolhido. Para confirmar de verdade, a consulta via CLI ao grupo inteiro é mais confiável e evita a falsa impressão de que o filtro não funciona.

---

### Rotação sob demanda exige Key ID, não aceita alias

**Sintoma:** `rotate-key-on-demand` falhou com `InvalidArnException` ao receber o alias.

**Causa:** a família de comandos de rotação atua sobre o material da chave e exige o Key ID (ou ARN), o mesmo comportamento do Lab 02.

**Solução:** resolver o alias antes com `describe-key --query KeyMetadata.KeyId`.

**Aprendizado:** operações que mexem no material da chave (rotação) não aceitam alias; operações de uso (encrypt/decrypt) aceitam.

---

## Detalhes Técnicos que Fizeram Diferença

- **Rotação automática mínima de 90 dias:** não dá para configurar um intervalo curto para ver a automática disparar. O teste real usa a rotação sob demanda, que ocorre na hora. A automática (`RotateKey`) gera um evento diferente da sob demanda (`RotateKeyOnDemand`).
- **Métrica sem dimensões + valor padrão 0:** com dimensão, a métrica só publica pontos quando há evento, o que desestabiliza o alarme. Sem dimensão e com padrão 0, a série é contínua e o alarme avalia de forma confiável.
- **Estados do alarme:** ao ser criado, o alarme passa por `INSUFFICIENT_DATA` → `OK` → (após rotação) `In alarm`. Essa progressão é normal.
- **Nomes distintos:** nomear o alarme igual ao tópico SNS gera confusão ao consultar recursos. Vale usar `kms-rotation-alarm` (alarme) e `kms-rotation-alerts` (tópico).
- **Custo:** a primeira trilha de eventos de gerenciamento não tem cobrança adicional de CloudTrail. O custo do lab fica no S3 e na ingestão do CloudWatch Logs. O Log Group vem com retenção "Nunca expirar", vale ajustar ou remover na limpeza.

---

## Referências

- [Monitor KMS keys with Amazon EventBridge](https://docs.aws.amazon.com/kms/latest/developerguide/kms-events.html)
- [Logging KMS API calls with CloudTrail](https://docs.aws.amazon.com/kms/latest/developerguide/logging-using-cloudtrail.html)
- [CloudWatch Logs metric filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html)
