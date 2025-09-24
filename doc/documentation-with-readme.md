# 📌 Seção para README.md — “Prompt para Copilot”

````markdown
## 🚀 Como gerar o arquivo ASL.json com Copilot

Este projeto já possui um `README.md` com as regras de conversão Camunda → Step Functions.  
Você pode usar o GitHub Copilot (ou outra IA) para gerar o arquivo `asl.json` a partir do BPMN.

### Passo-a-passo
1. Abra o BPMN do processo em `bpmn/SEU_PROCESSO.bpmn`.
2. Abra este `README.md` (Copilot consegue usar as instruções como contexto).
3. No Copilot Chat, cole o **prompt padronizado** abaixo (ajuste `SEU_PROCESSO` para o nome real).
4. Copie o JSON de saída e salve em `migration/asl/SEU_PROCESSO.asl.json`.
5. Valide localmente:
   ```bash
   npm install -g asl-validator@latest
   asl-validator migration/asl/SEU_PROCESSO.asl.json --quiet
   node scripts/lint-asl.js migration/asl
````

6. Abra PR. O pipeline do GitHub Actions repetirá as validações.

---

### 📝 Prompt padronizado para Copilot

Cole exatamente isto no chat:

```
Você é responsável por gerar o arquivo Step Functions (ASL.json) a partir de um BPMN.

Entrada:
- O arquivo BPMN está em `bpmn/SEU_PROCESSO.bpmn`
- O repositório contém um `README.md` com todas as regras de conversão.

Tarefas:
1. Leia as instruções do README.md e siga à risca.
2. Converta cada ServiceTask em um estado `Task` do tipo `arn:aws:states:::lambda:invoke`.
   - Use `FunctionName` a partir do `x-aws:lambdaLogicalName` no BPMN.
   - Sempre inclua `Qualifier: "$LATEST"`.
   - Payload deve ser `Payload.$ = "$"`.
   - Inclua `TimeoutSeconds` e `ResultPath` obrigatoriamente.
   - Use `ResultSelector` para mapear `$.Payload` para a chave definida.
   - Adicione `Retry` do catálogo `camunda_R5_PT1M` se não houver override.
3. ExclusiveGateway deve virar `Choice` com `Default` obrigatório.
4. ParallelGateway deve virar `Parallel` com cada branch terminando em `"End": true`.
5. Use nomes funcionais dos estados (sem prefixos técnicos).
6. Finalize sempre com `FIM_Sucesso` e `FIM_Falha`.

Saída:
- Gere apenas **um JSON válido de Step Functions** (ASL) pronto para o `asl-validator`.
- Salve no caminho `migration/asl/SEU_PROCESSO.asl.json`.
- Não adicione comentários, texto fora do JSON ou explicações.
```

---

### ✅ Checklist de qualidade antes do PR

* [ ] Toda Task tem `TimeoutSeconds` e `ResultPath`.
* [ ] Choices têm `Default`.
* [ ] Parallel tem branches com `"End": true`.
* [ ] Estados finais são `FIM_Sucesso` e `FIM_Falha`.
* [ ] Passou no `asl-validator` e no `scripts/lint-asl.js`.
