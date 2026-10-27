# Lições Aprendidas

## O Que Deu Certo

1. **Compatibilidade retroativa automática**: cada dado permanece decifrável pela versão de material que o criou, e o KMS guarda todas elas.

2. **A identidade da chave é preservada**: Key ID, ARN, alias e Key Policy não mudam, e as aplicações seguem funcionando sem alteração de código ou configuração.

3. **O `KeyMaterialId` mostra a versão em uso**: comparar o `KeyMaterialId` retornado no decrypt de dados antigos e novos mostrou qual versão do material foi usada em cada operação.

4. **Rotação sob demanda**: permite renovar o material criptográfico na hora, sem esperar o ciclo anual. Útil tanto para responder a uma suspeita de comprometimento quanto para validar o processo de rotação em ambiente controlado.

5. **Auditoria via CloudTrail**: o evento `RotateKeyOnDemand` é registrado automaticamente, fornecendo evidência para compliance sem configuração adicional.

---

## O Que Deu Errado e Como Resolvi

### A família de comandos de rotação não aceita alias

**Sintoma:** ao usar o alias em comandos de rotação, todos falharam com o mesmo erro:

```
An error occurred (InvalidArnException): Key Aliases are not supported for this operation.
```

**Causa:** os comandos que atuam diretamente sobre o material da chave exigem o Key ID (ou o ARN). O alias, que é um ponteiro para a chave, não é aceito nesses casos. Testando um a um, o padrão ficou claro:

| Comando | Aceita alias? |
|---------|:-------------:|
| `describe-key` | ✅ Sim |
| `enable-key-rotation` | ❌ Só Key ID |
| `get-key-rotation-status` | ❌ Só Key ID |
| `rotate-key-on-demand` | ❌ Só Key ID |
| `list-key-rotations` | ❌ Só Key ID |

**Aprendizado:** nem todos os comandos KMS tratam alias e Key ID como equivalentes. Operações de uso (Encrypt, Decrypt) aceitam alias, mas a família de rotação exige o Key ID. Vale resolver o alias para o Key ID antes de rodar esses comandos.

---

### `AccessDeniedException` ao consultar rotações pela EC2

**Sintoma:** ao tentar `describe-key` e `list-key-rotations` de dentro da instância, o acesso foi negado:

```
AccessDeniedException ... EC2KMSLabRole ... is not authorized to perform: kms:ListKeyRotations
```

**Causa:** a role `EC2KMSLabRole` (definida no Lab 01) tem apenas `kms:GenerateDataKey`, `kms:Encrypt` e `kms:Decrypt`. Consultar rotações é uma operação administrativa, fora do escopo da instância.

**Aprendizado:** o erro é, na verdade, o menor privilégio funcionando. A instância consegue usar a chave para criptografar e descriptografar, mas não consegue inspecionar a administração dela. Operações administrativas (como listar rotações) devem ser feitas com credenciais de administrador, não pela role da aplicação.

---

## Conceitos que Ficaram Claros

1. **Rotação não é re-criptografia**: os dados antigos permanecem com o material original. Apenas novas operações de criptografia usam o material novo. A rotação limita o escopo temporal de um comprometimento, mas não "atualiza" dados existentes.

2. **A rotação tem um limite**: ela protege contra o comprometimento do material criptográfico. Se o acesso à chave em si for comprometido (a role ou a policy), rotação não resolve. Nesse caso, o caminho é desativar a chave e criar uma nova.

---

## Referências

- [Key Rotation Concepts](https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html)
- [On-demand Rotation](https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys-on-demand.html)
- [CloudTrail Events for KMS](https://docs.aws.amazon.com/kms/latest/developerguide/logging-using-cloudtrail.html)
