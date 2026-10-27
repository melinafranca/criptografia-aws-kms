# Lições Aprendidas

## O Que Deu Certo

1. **Descriptografia cross-region transparente**: um arquivo criptografado em `us-east-1` foi descriptografado em `eu-west-1` só trocando o `--region`, sem nenhuma etapa intermediária. A réplica usou o mesmo material da primária.

2. **Fluxo bidirecional confirmado**: o teste inverso (criptografar na réplica em `eu-west-1` e descriptografar na primária em `us-east-1`) também funcionou. Não há uma relação fixa de origem e destino: as duas chaves são funcionalmente equivalentes.

3. **Encryption Context validado em qualquer região**: descriptografar sem o context, ou com um context errado, falhou com `InvalidCiphertextException` mesmo rodando em `eu-west-1`. A camada de segurança do context acompanha o dado, independente da região.

4. **CloudShell para operação cross-region**: rodar tudo no CloudShell e alternar entre regiões só pelo `--region` deixou a demonstração simples e independente de qualquer instância EC2.

---

## O Que Deu Errado e Como Resolvi

### O alias não é replicado junto com a chave

**Sintoma:** ao criar a réplica via CLI, o material criptográfico foi replicado, mas eu não tinha certeza se o alias também iria para a segunda região. Precisava confirmar antes de documentar.

**Investigação:** criei um alias de teste apenas em `us-east-1` e verifiquei se ele aparecia sozinho em `eu-west-1`:

```bash
aws kms create-alias --alias-name alias/testeSyncCLI --target-key-id <INSIRA_O_KEY_ID> --region us-east-1
```

```bash
aws kms list-aliases --region eu-west-1 --query "Aliases[?AliasName=='alias/testeSyncCLI']"
```

O retorno foi `[]`: o alias não apareceu em `eu-west-1`.

**Causa:** a AWS sincroniza apenas as propriedades compartilhadas entre chaves multi-região (como o material criptográfico). Aliases, tags e Key Policy podem diferir por região e não são replicados.

**Aprendizado:** o material é replicado, mas o alias precisa ser criado em cada região. No console, o campo de alias vem sugerido durante a criação da réplica (fácil de confundir com replicação automática). Via CLI, o `replicate-key` não cria alias nenhum: é necessário um `create-alias` na segunda região.

---

### Colisão de alias entre console e CLI

**Sintoma:** ao reproduzir o lab via CLI depois de já ter feito pelo console, o `create-alias` falhou:

```
An error occurred (AlreadyExistsException) when calling the CreateAlias operation: An alias with the name alias/myMultiRegionKey already exists
```

**Causa:** o alias criado no fluxo do console já ocupava esse nome na mesma conta e região. Alias é um recurso único por região: não é possível ter dois com o mesmo nome apontando para chaves diferentes.

**Solução:** para validar os comandos sem conflito, usei um alias distinto na sessão de CLI (`alias/myMultiRegionKeyCLI`). Em um ambiente limpo, feito só pela CLI, o nome padrão funcionaria sem colisão.

**Aprendizado:** console e CLI operam sobre os mesmos recursos da conta. Reproduzir um lab pelas duas vias na mesma conta pode gerar colisões de nome e recursos duplicados. Vale planejar a limpeza para não pagar por chaves repetidas.

---

### A réplica não fica pronta instantaneamente

**Sintoma:** logo após o `replicate-key`, a réplica retornou com `"KeyState": "Creating"` e `"Enabled": false`.

**Causa:** a criação da réplica leva alguns segundos. Usá-la nesse intervalo, antes de ficar `Enabled`, resultaria em falha.

**Solução:** confirmar o estado antes de usar a chave na segunda região:

```bash
aws kms describe-key --key-id alias/myMultiRegionKey --query KeyMetadata.KeyState --output text --region eu-west-1
```

**Aprendizado:** replicação não é instantânea. Em automações, vale aguardar o estado `Enabled` antes de encaminhar operações para a réplica.

---

## Referências

- [Multi-Region Keys Overview](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-overview.html)
- [Replicating Multi-Region Keys](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-replicate.html)
- [Managing Multi-Region Keys](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-manage.html)
