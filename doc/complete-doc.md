# Artefatos iniciais — Camunda 7 → AWS Step Functions

Abaixo estão **dois arquivos prontos** para colocar no seu repositório:

1. `migration/migration_contract.yaml` — contrato **máquina‑legível** com regras e políticas (v1).
2. `.github/workflows/migrate.yml` — pipeline **GitHub Actions** para transpilar, validar e (opcional) enriquecer via IA.

Aplique os ajustes marcados como `TODO:` / `REVIEW:` e faça um PR com esses dois arquivos.

---

## `migration/migration_contract.yaml`

```yaml
# v1 — Contrato de Migração Camunda 7 → AWS Step Functions (ASL)
# Este arquivo é lido pelo transpiler (bpmn2asl) e pelos lints corporativos.
# Qualquer alteração deve passar por revisão do time de Arquitetura.

version: 1
meta:
  owner_team: "time-arquitetura"            # TODO: ajuste
  contact: "arquitetura@example.com"        # TODO: ajuste
  region: "sa-east-1"
  allow_overrides:
    # Quais chaves os repositórios de processos podem sobrescrever localmente
    - policies.default_timeouts
    - policies.retry_catalog
    - security.mask

naming:
  normalize: true
  state_prefixes:
    task: "T"
    choice: "G"
    map: "M"
    parallel: "P"
    wait: "W"
    end_success: "FIM_Sucesso"
    end_failure: "FIM_Falha"

policies:
  default_timeouts:
    task_seconds: 30
  retry_catalog:
    idempotent_default:
      - { ErrorEquals: ["States.Timeout"], IntervalSeconds: 2, MaxAttempts: 2, BackoffRate: 2.0 }
      - { ErrorEquals: ["States.ALL"],     IntervalSeconds: 3, MaxAttempts: 3, BackoffRate: 2.0 }
    non_idempotent_default:
      - { ErrorEquals: ["States.Timeout"], IntervalSeconds: 5, MaxAttempts: 1 }
  error_taxonomy:
    functional: ["Dominio.*", "Validacao.*"]
    technical:  ["States.*", "Infra.*"]

mapping:
  serviceTask:
    type: task            # sempre Task no ASL
    resource: "lambda"    # outras opções futuras: activity, sqs, sns
    resource_ref: "$prop:x-aws:lambdaLogicalName"
    timeoutSeconds: "$prop:x-aws:timeoutSeconds|policies.default_timeouts.task_seconds"
    resultPath: "$prop:x-aws:resultPath|$.result"
    parameters:
      # Como o payload é enviado para a Lambda (ajuste conforme padrão)
      payload_template: "$"  # envia o input inteiro por padrão
    retry: "$prop:x-aws:retry|policies.retry_catalog.idempotent_default"
    catch:
      - { ErrorEquals: ["Dominio.*"], Next: "G_RoteiaErroFuncional" }
      - { ErrorEquals: ["States.ALL"], Next: "G_RoteiaErroTecnico" }

  exclusiveGateway:
    type: choice
    default: "$prop:x-aws:default|FIM_Falha"
    expr_language: "jsonpath"   # FEEL/EL serão traduzidos para JSONPath/JSONata

  parallelGateway:
    type: parallel

  userTask:
    type: callback
    pattern: "taskToken"        # Callback Pattern (SQS/SNS/EventBridge)

  timerEvent:
    type: wait
    mode: "$prop:x-aws:waitMode|seconds"  # seconds|timestamp

  callActivity:
    type: subflow
    mode: "$prop:x-aws:subflow|sync"      # sync|async
    arn: "$prop:x-aws:subflowArn|"        # opcional: indicar ARN alvo

security:
  mask:
    input:  ["$.cpf", "$.cartao.numero"]
    output: ["$.pessoa.cpf", "$.token"]
  kms_key_arn: "arn:aws:kms:sa-east-1:111111111111:key/REVIEW-KEY"  # TODO

validation:
  require_default_on_choice: true
  require_timeouts_on_task: true
  require_resultpath_explicit: true
  require_mask_for_pii: true

postprocess_ai:
  enabled: true                 # pode ser desligado por repo
  provider: "bedrock"
  model: "anthropic.claude-3-5" # REVIEW: escolha corporativa
  allow_changes:
    - naming                   # normalizar nomes (ex.: T1_..., G1_...)
    - comments                 # preencher Comment/Description
    - expr_translation         # FEEL/EL → JSONPath/JSONata
  forbidden_changes:
    - graph                    # IA não pode alterar o grafo/fluxo
    - modes                    # sync/async, callback vs. task
```

---

## `.github/workflows/migrate.yml`

```yaml
name: Camunda → Step Functions (Transpile & Validate)

on:
  push:
    paths:
      - 'bpmn/**/*.bpmn'
      - 'migration/migration_contract.yaml'
  pull_request:
    paths:
      - 'bpmn/**/*.bpmn'
      - 'migration/migration_contract.yaml'

jobs:
  migrate:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # para OIDC (assume-role na AWS)
      actions: read
      pull-requests: write

    env:
      AWS_REGION: sa-east-1
      CONTRACT_FILE: migration/migration_contract.yaml
      OUT_DIR: migration/asl

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Java 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Setup Node (para validador ASL)
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Prepare output dir
        run: |
          mkdir -p $OUT_DIR

      - name: Transpile BPMN → ASL (bpmn2asl)
        # Substitua pelo seu runner (jar, npm, container). Exemplo usando um jar versionado no repositório de ferramentas.
        run: |
          echo "::group::Transpiling"
          for f in bpmn/*.bpmn; do
            base=$(basename "$f" .bpmn)
            java -jar tools/bpmn2asl.jar \
              --bpmn "$f" \
              --contract "$CONTRACT_FILE" \
              --out "$OUT_DIR/$base.asl.json"
          done
          echo "::endgroup::"

      - name: Validar ASL (schema)
        run: |
          npm install -g asl-validator@latest
          for f in $OUT_DIR/*.asl.json; do
            echo "Validando $f"; asl-validator "$f" --quiet
          done

      - name: Lints corporativos
        # Substitua pelo container/ação oficial da sua org, se houver
        run: |
          node scripts/lint-asl.js $OUT_DIR || true  # TODO: implementar lints

      - name: Configure AWS credentials (OIDC)
        if: ${{ hashFiles('**/*.asl.json') != '' }}
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/gh-actions-stepfunctions   # TODO
          aws-region: ${{ env.AWS_REGION }}

      - name: (Opcional) Pós-processamento IA via Lambda wrapper
        if: ${{ env.RUN_AI == 'true' }}
        run: |
          for f in $OUT_DIR/*.asl.json; do
            out_tmp=${f%.json}.ai.json
            aws lambda invoke \
              --function-name asl-postprocess \
              --payload "$(jq -c -n --argfile def $f '{definition:$def, contract:input.CONTRACT}')" \
              --cli-binary-format raw-in-base64-out \
              /dev/stdout | jq -r '.Payload' > "$out_tmp"
            mv "$out_tmp" "$f"
          done

      - name: Revalidar ASL pós-IA
        if: ${{ env.RUN_AI == 'true' }}
        run: |
          for f in $OUT_DIR/*.asl.json; do
            asl-validator "$f" --quiet
          done

      - name: Publicar artefatos
        uses: actions/upload-artifact@v4
        with:
          name: asl-json
          path: ${{ env.OUT_DIR }}/*.asl.json

      - name: (Opcional) Abrir PR no repo de IaC
        if: ${{ env.OPEN_IAC_PR == 'true' }}
        run: |
          echo "TODO: implementar sync para repositório de IaC (CDK/Terraform)"
```

---

## Dicas finais

* **README + Contrato**: mantenha ambos. O README instrui pessoas; o contrato guia ferramentas/IA.
* **Papéis & permissões**: crie uma role dedicada a partir de OIDC para o workflow (`gh-actions-stepfunctions`).
* **Validação é obrigatória**: schema + lints. A IA é opcional e não pode mudar o grafo.
* **Padrões evolutivos**: versione este contrato (v1, v1.1…). Anexe changelog para os times.

---

# Exemplos “Golden” para validar o pipeline

Abaixo seguem um **BPMN anotado** e o **ASL esperado**. Use-os como teste de fumaça: ao abrir um PR com este `.bpmn`, o pipeline deve gerar exatamente o `.asl.json` abaixo (ou semanticamente idêntico). Qualquer divergência indica quebra de determinismo ou de regra do contrato.

## `bpmn/exemplo-processo.bpmn`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
             xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
             xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC"
             xmlns:omgdi="http://www.omg.org/spec/DD/20100524/DI"
             xmlns:camunda="http://camunda.org/schema/1.0/bpmn"
             targetNamespace="http://example.com/bpmn">
  <process id="processo_exemplo" name="Processo Exemplo" isExecutable="true">

    <startEvent id="start" name="Start">
      <outgoing>flow_start_to_t1</outgoing>
    </startEvent>

    <serviceTask id="t1_validar" name="Validar CPF" camunda:type="external">
      <extensionElements>
        <camunda:properties>
          <camunda:property name="x-aws:lambdaLogicalName" value="FnValidarCPF" />
          <camunda:property name="x-aws:timeoutSeconds" value="10" />
          <camunda:property name="x-aws:resultPath" value="$.validacao" />
          <!-- opcional: política custom de retry; se ausente, aplica catálogo do contrato -->
          <!-- <camunda:property name="x-aws:retry" value='[{"ErrorEquals":["States.Timeout"],"IntervalSeconds":2,"MaxAttempts":2}]' /> -->
        </camunda:properties>
      </extensionElements>
      <incoming>flow_start_to_t1</incoming>
      <outgoing>flow_t1_to_g1</outgoing>
    </serviceTask>

    <exclusiveGateway id="g1_roteia" name="Roteia">
      <extensionElements>
        <camunda:properties>
          <camunda:property name="x-aws:default" value="fim_falha" />
          <!-- Exemplo de expressão; se vier em FEEL/EL, o pós-processo IA pode traduzir para JSONPath/JSONata -->
          <camunda:property name="x-aws:expr" value="$.validacao.aprovado == true" />
        </camunda:properties>
      </extensionElements>
      <incoming>flow_t1_to_g1</incoming>
      <outgoing>flow_g1_to_t2</outgoing>
      <outgoing>flow_g1_to_fim_falha</outgoing>
    </exclusiveGateway>

    <serviceTask id="t2_processar" name="Processar" camunda:type="external">
      <extensionElements>
        <camunda:properties>
          <camunda:property name="x-aws:lambdaLogicalName" value="FnProcessar" />
          <camunda:property name="x-aws:timeoutSeconds" value="20" />
          <camunda:property name="x-aws:resultPath" value="$.processamento" />
        </camunda:properties>
      </extensionElements>
      <incoming>flow_g1_to_t2</incoming>
      <outgoing>flow_t2_to_fim_sucesso</outgoing>
    </serviceTask>

    <endEvent id="fim_sucesso" name="Fim Sucesso">
      <incoming>flow_t2_to_fim_sucesso</incoming>
    </endEvent>

    <endEvent id="fim_falha" name="Fim Falha">
      <incoming>flow_g1_to_fim_falha</incoming>
    </endEvent>

    <!-- Flows -->
    <sequenceFlow id="flow_start_to_t1" sourceRef="start" targetRef="t1_validar" />
    <sequenceFlow id="flow_t1_to_g1" sourceRef="t1_validar" targetRef="g1_roteia" />
    <sequenceFlow id="flow_g1_to_t2" sourceRef="g1_roteia" targetRef="t2_processar" />
    <sequenceFlow id="flow_g1_to_fim_falha" sourceRef="g1_roteia" targetRef="fim_falha" />
    <sequenceFlow id="flow_t2_to_fim_sucesso" sourceRef="t2_processar" targetRef="fim_sucesso" />

  </process>
</definitions>
```

## `migration/asl/exemplo-processo.asl.json` (esperado)

```json
{
  "Comment": "Migrado de Camunda 7 — Processo Exemplo",
  "StartAt": "T1_ValidarCPF",
  "States": {
    "T1_ValidarCPF": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:sa-east-1:111111111111:function:FnValidarCPF",
      "TimeoutSeconds": 10,
      "ResultPath": "$.validacao",
      "Retry": [
        { "ErrorEquals": ["States.Timeout"], "IntervalSeconds": 2, "MaxAttempts": 2, "BackoffRate": 2.0 },
        { "ErrorEquals": ["States.ALL"], "IntervalSeconds": 3, "MaxAttempts": 3, "BackoffRate": 2.0 }
      ],
      "Catch": [
        { "ErrorEquals": ["Dominio.*"], "Next": "G1_RoteiaErroFuncional" },
        { "ErrorEquals": ["States.ALL"], "Next": "G1_RoteiaErroTecnico" }
      ],
      "Next": "G1_Roteia"
    },
    "G1_Roteia": {
      "Type": "Choice",
      "Choices": [
        { "Variable": "$.validacao.aprovado", "BooleanEquals": true, "Next": "T2_Processar" }
      ],
      "Default": "FIM_Falha"
    },
    "T2_Processar": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:sa-east-1:111111111111:function:FnProcessar",
      "TimeoutSeconds": 20,
      "ResultPath": "$.processamento",
      "Retry": [
        { "ErrorEquals": ["States.Timeout"], "IntervalSeconds": 2, "MaxAttempts": 2, "BackoffRate": 2.0 },
        { "ErrorEquals": ["States.ALL"], "IntervalSeconds": 3, "MaxAttempts": 3, "BackoffRate": 2.0 }
      ],
      "Next": "FIM_Sucesso"
    },
    "G1_RoteiaErroFuncional": { "Type": "Fail", "Error": "Dominio.Erro" },
    "G1_RoteiaErroTecnico":  { "Type": "Fail", "Error": "Infra.Erro" },
    "FIM_Sucesso": { "Type": "Succeed" },
    "FIM_Falha":   { "Type": "Fail", "Error": "Fluxo.Reprovado" }
  }
}
```

## Source Map sugerido (opcional)

Crie `migration/docs/map.json` para rastreabilidade (útil em logs/observabilidade):

```json
{
  "bpmnIdToAsl": {
    "t1_validar": "T1_ValidarCPF",
    "g1_roteia": "G1_Roteia",
    "t2_processar": "T2_Processar",
    "fim_sucesso": "FIM_Sucesso",
    "fim_falha": "FIM_Falha"
  }
}
```

## Lint mínimo (stub) — `scripts/lint-asl.js`

> Stub simples para começar; substitua pela ação oficial quando disponível.

```js
#!/usr/bin/env node
const fs = require('fs');
const path = process.argv[2];
if (!path) { console.error('Uso: node lint-asl.js <dir>'); process.exit(2); }

let errors = 0;
for (const f of fs.readdirSync(path)) {
  if (!f.endsWith('.json')) continue;
  const def = JSON.parse(fs.readFileSync(`${path}/${f}`, 'utf8'));
  const states = def.States || {};
  // Regra: toda Task tem TimeoutSeconds e ResultPath explícito
  for (const [name, st] of Object.entries(states)) {
    if (st.Type === 'Task') {
      if (typeof st.TimeoutSeconds !== 'number') {
        console.error(`[${f}] Task ${name} sem TimeoutSeconds`); errors++; }
      if (typeof st.ResultPath === 'undefined') {
        console.error(`[${f}] Task ${name} sem ResultPath`); errors++; }
    }
    if (st.Type === 'Choice') {
      if (!('Default' in st)) { console.error(`[${f}] Choice ${name} sem Default`); errors++; }
    }
  }
}
process.exit(errors ? 1 : 0);
```
