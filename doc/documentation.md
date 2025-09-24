# Quickstart — do zero ao `asl.json` determinístico

## 0) Pré-requisitos (local)

* **Java 17** (para rodar o transpiler `.jar`)
* **Node 20** + **npm** (para `asl-validator` e lints)
* (Opcional) **AWS CLI** configurado se for implantar em DEV depois

## 1) Estruture o repositório do processo

No repositório Camunda (um por processo), crie as pastas base:

```bash
mkdir -p bpmn migration/asl migration/docs scripts tools .github/workflows
```

> Dica: se a empresa for usar um **template de repositório**, inclua essas pastas e arquivos no template para todos os times.

## 2) Adicione os arquivos de governança

Coloque estes **dois arquivos** (eles já estão prontos no canvas desta conversa):

* `migration/migration_contract.yaml`
* `.github/workflows/migrate.yml`

E **crie** o lint mínimo:

```bash
cat > scripts/lint-asl.js <<'EOF'
#!/usr/bin/env node
const fs = require('fs');
const dir = process.argv[2]; if (!dir) { console.error('Uso: node lint-asl.js <dir>'); process.exit(2); }
let errors = 0;
for (const f of fs.readdirSync(dir)) {
  if (!f.endsWith('.json')) continue;
  const def = JSON.parse(fs.readFileSync(`${dir}/${f}`, 'utf8'));
  const states = def.States || {};
  for (const [name, st] of Object.entries(states)) {
    if (st.Type === 'Task') {
      if (typeof st.TimeoutSeconds !== 'number') { console.error(`[${f}] Task ${name} sem TimeoutSeconds`); errors++; }
      if (typeof st.ResultPath === 'undefined') { console.error(`[${f}] Task ${name} sem ResultPath`); errors++; }
    }
    if (st.Type === 'Choice' && !('Default' in st)) {
      console.error(`[${f}] Choice ${name} sem Default`); errors++;
    }
  }
}
process.exit(errors ? 1 : 0);
EOF
chmod +x scripts/lint-asl.js
```

> **Ajustes obrigatórios** no `migration_contract.yaml`:
>
> * `kms_key_arn` (chave KMS da sua conta)
> * (Opcional) desligar IA por padrão (`postprocess_ai.enabled: false`) se ainda não tiver a Lambda de pós-processo
> * (Opcional) remover prefixos `T/G/P` do `naming` se não quiser usar

## 3) Coloque seu(s) BPMN(s)

Salve os arquivos Camunda em `bpmn/`.
**Anote as ServiceTasks** com `camunda:properties` (ex.: `x-aws:lambdaLogicalName`, `x-aws:timeoutSeconds`, `x-aws:resultPath`).
Para Gateways exclusivos, inclua `x-aws:default` e, quando tiver condição, `x-aws:expr` (JSONPath/JSONata).

> Se quiser um exemplo, há um **BPMN de exemplo** no canvas para copiar e adaptar.

## 4) Transpiler (BPMN → ASL)

Coloque o binário do transpiler em `tools/bpmn2asl.jar` (da sua forja interna).
Então rode:

```bash
# 4.1 gerar ASL para cada .bpmn
for f in bpmn/*.bpmn; do
  base=$(basename "$f" .bpmn)
  java -jar tools/bpmn2asl.jar \
    --bpmn "$f" \
    --contract migration/migration_contract.yaml \
    --out "migration/asl/$base.asl.json"
done

# 4.2 instalar validador ASL
npm i -g asl-validator@latest

# 4.3 validar schema
for f in migration/asl/*.asl.json; do
  echo "Validando $f"
  asl-validator "$f" --quiet
done

# 4.4 lints corporativos
node scripts/lint-asl.js migration/asl
```

Se qualquer etapa falhar, **corrija o BPMN** (metadados ausentes) ou **ajuste o contrato** (se for permitido) e rode de novo.

## 5) Teste de **determinismo**

Rode o transpiler **duas vezes** e verifique que nada mudou:

```bash
git add -A && git commit -m "Primeira geração ASL"
# gere de novo (repita o passo 4)
git diff -- migration/asl
# deve não exibir diferenças
```

Se aparecer diff, há algo não determinístico (ex.: ordem de arrays, timestamp, ID aleatório). Corrija no transpiler/lint/contrato.

## 6) CI no GitHub (padronizado)

Ao fazer **push** ou **PR** alterando `bpmn/*.bpmn` ou o contrato, o workflow em `.github/workflows/migrate.yml` já:

1. Faz **checkout**
2. Roda **transpiler**
3. Valida com **asl-validator**
4. Executa **lints corporativos**
5. (Opcional) roda **IA** via Lambda `asl-postprocess` (se você definir `RUN_AI=true`)
6. Publica os artefatos em `Actions → Artifacts`

### O que você precisa configurar no CI (uma vez por org):

* Criar a **IAM role** para OIDC (`gh-actions-stepfunctions`) e colocar o ARN no workflow.
* Decidir se a etapa de **IA** ficará desligada por padrão (`RUN_AI=false`).
* (Quando quiser implantar) acione outro workflow que cria/atualiza a state machine via IaC (CDK/Terraform) com o `asl.json` como fonte.

---

# Como “encaixar” tudo em centenas de repositórios

## A) Template de repositório

Crie um **repo template corporativo** contendo:

* `README.md` (instruções de migração)
* `migration/migration_contract.yaml` (versão v1 da empresa)
* `.github/workflows/migrate.yml`
* `scripts/lint-asl.js`
* (Opcional) `bpmn/exemplo-processo.bpmn` como golden

Times criam seus repositórios **a partir desse template**.

## B) Regras centralizadas e evolutivas

* Mantenha um **catálogo raiz** (outro repo) com o `migration_contract.yaml` **v1/v1.1/v2…**
* Nos projetos, permita **apenas** overrides específicos (já listados em `allow_overrides` no contrato)
* Toda evolução de regra passa por **PR de Arquitetura** e **changelog**.

## C) Reusable Workflow (opcional)

Transforme o `.github/workflows/migrate.yml` em um **reusable workflow** no repositório de plataforma.
Nos projetos, só referencie com:

```yaml
jobs:
  migrate:
    uses: your-org/platform/.github/workflows/migrate.yml@v1
```

Isso garante uniformidade sem copiar/colar.

---

# Perguntas rápidas

**“O que é ‘Este arquivo é lido pelo transpiler e pelos lints’?”**
Quer dizer que o `migration_contract.yaml` é **contrato executável**:

* o **transpiler** usa as regras para gerar o ASL;
* os **lints** usam as mesmas regras para validar o que foi gerado.

**“E se meu BPMN não tiver `x-aws:lambdaLogicalName`?”**
O transpiler deve **falhar** com mensagem clara. Não invente ARN. Preencha as `camunda:properties`.

**“Posso usar JSONata?”**
Pode, mas **só quando necessário** e de forma localizada; o padrão recomendado é **ASL clássico** usando `Parameters`, `ResultSelector`, `ResultPath`, por ser mais **lintável e determinístico**.

---

# Checagem final (antes de aprovar PR)

* ✅ Todas as Tasks têm `TimeoutSeconds` e `ResultPath`
* ✅ Choices têm `Default`
* ✅ Retry aplicado do **catálogo** (ex.: `camunda_R5_PT1M`)
* ✅ Campos PII mascarados conforme `security.mask`
* ✅ `asl-validator` OK e `scripts/lint-asl.js` OK
* ✅ Regerar não muda nada (`git diff` vazio)

---
