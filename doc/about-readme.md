Ótima pergunta — existem **dois nomes oficiais** (e um “extra”) que costumam aparecer quando a gente fala de “um README que serve de receita para o Copilot”:

1. **`.github/copilot-instructions.md`** (o “README do Copilot”)
   É um arquivo **Markdown** com instruções em linguagem natural que orienta o Copilot sobre **como trabalhar no seu repositório**: visão geral do projeto, padrões de código, como buildar/testar, convenções de commits, guias de segurança, etc. Ele é lido automaticamente pelo Copilot (Chat, Code Review e Coding Agent) e funciona como a “receita” principal do repo. ([GitHub Docs][1])

2. **`copilot-setup-steps.yml`** (a “receita em YAML”)
   É um workflow **GitHub Actions** especial, com um único job chamado `copilot-setup-steps`, que roda **antes** do Copilot começar uma tarefa do **Copilot Coding Agent**. Você descreve em YAML o que precisa para preparar o ambiente: instalar SDKs, restaurar cache, semear variáveis, gerar artefatos, etc. Pense nele como uma “playbook” de provisionamento/bootstrapping para o agente. ([GitHub Docs][2])

> Extra (VS Code): **Prompt Files** e `AGENTS.md`
> No VS Code você pode ter **Prompt Files** (p.ex., um arquivo com prompts prontos) e até um **`AGENTS.md`** experimental para instruções multi-agente. Mas, para repositório GitHub, o nome canônico continua sendo **`copilot-instructions.md`**. ([Visual Studio Code][3])

---

# O que colocar em cada um

## `copilot-instructions.md` (Markdown)

* **Objetivo do projeto** e escopo.
* **Stack, versões e padrões** (ex.: Java 17, Quarkus, Sonar rules, OWASP).
* **Regras de design/arquitetura** (DDD, camadas, eventos, idempotência).
* **Como rodar/buildar/testar** (mvn/gradle, containers, Testcontainers).
* **Guia de estilo e revisões** (lint, formatação, convenções de PR).
* **Políticas de segurança** (KMS, secrets, PII, logs mascarados).
* **Exemplos breves** do que é “bom” vs “ruim”.
  Essas instruções passam a **groundear** as respostas do Copilot ao gerar código, revisar PR e conversar sobre o repo. ([GitHub Docs][4])

## `copilot-setup-steps.yml` (YAML)

* Um único job `copilot-setup-steps`.
* Passos para preparar o ambiente do agente:

  * Checar o código, configurar JDK/Node, baixar deps, criar `.env.example`, semear banco de teste, etc.
* **Importante:** precisa estar no **branch padrão** para disparar. ([GitHub Docs][2])

---

# Exemplo rápido

**`.github/copilot-instructions.md`**

```md
# Instruções do Copilot (Repositório ACME)

- Linguagem/stack: Java 17 + Quarkus; testes com JUnit5 + Testcontainers.
- Padrões: DDD (módulos domain/app/infra), eventos imutáveis (CloudEvents),
  logs sem dados sensíveis (mascarar CPF), checagem Sonar antes do merge.
- Build: `./mvnw clean verify -Pnative` para binário nativo.
- Estilo: Lombok permitido; evitar lógica em controllers; preferir Records para DTOs.
- Segurança: nunca logar segredos; usar KMS para chaves; validar input externo.
- Ao criar novas Lambdas, usar handler padrão, timeouts < 15s, e idempotência via chave.
```

**`.github/workflows/copilot-setup-steps.yml`**

```yaml
name: copilot-setup-steps
on:
  workflow_dispatch:
jobs:
  copilot-setup-steps:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: 'temurin', java-version: '17' }
      - name: Build & test
        run: ./mvnw -q -B -DskipITs verify
```

([GitHub Docs][2])

---

# Boas práticas para sua apresentação

* **Seja específico e conciso**: listas curtas e prescritivas funcionam melhor do que textos longos. ([The GitHub Blog][5])
* **Coloque o que o Copilot precisa para agir**: comandos de build/test, padrões obrigatórios e links para docs internas. ([GitHub Docs][4])
* **Não duplique**: se algo muda sempre (por ex., versão da JDK), mantenha num único lugar e aponte no `copilot-instructions.md`.
* **Evite conteúdo sensível**: não cole segredos; descreva políticas (ex.: “usar Secret Manager”).
* **Valide o efeito**: abra o Copilot Chat no repo e peça: “@workspace leia as instruções e gere um esqueleto de microserviço com nossos padrões”. (Dica de uso de contexto) ([Stack Overflow][6])

---

# Limitações e pegadinhas

* **Contexto não é infinito**: o Copilot resume/seleciona partes do repo; escrever instruções bem objetivas ajuda. ([Stack Overflow][6])
* **Atualize sempre**: se o padrão mudar (ex.: “usar Step Functions Callback”), atualize o `copilot-instructions.md` para evitar drift. ([The GitHub Blog][5])
* **Branch padrão**: o `copilot-setup-steps.yml` só dispara se estiver presente no **default branch**. ([GitHub Docs][2])

---

## TL;DR para o slide:

* **Nome do “README-receita”:** `.github/copilot-instructions.md`.
* **Receita em YAML para preparar ambiente:** `copilot-setup-steps.yml` (GitHub Actions, job único).
* **Onde funciona:** Copilot Chat, Code Review e Coding Agent consideram as instruções do repo.
* **Conteúdo ideal:** stack + padrões + como buildar/testar + regras de segurança + exemplos.
* **Ganhos:** respostas mais consistentes, geração de código alinhada ao padrão da empresa e onboarding acelerado. ([GitHub Docs][1])

Se quiser, eu já te entrego um **modelo pronto** de `copilot-instructions.md` adaptado ao seu padrão (Java 17, Quarkus, AWS, Step Functions, SQS, KMS, logs mascarados, etc.).

[1]: https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot?utm_source=chatgpt.com "Adding repository custom instructions for GitHub Copilot"
[2]: https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/customize-the-agent-environment?utm_source=chatgpt.com "About customizing Copilot coding agent's development ..."
[3]: https://code.visualstudio.com/docs/copilot/customization/custom-instructions?utm_source=chatgpt.com "Use custom instructions in VS Code"
[4]: https://docs.github.com/copilot/how-tos/agents/copilot-coding-agent/best-practices-for-using-copilot-to-work-on-tasks?utm_source=chatgpt.com "Best practices for using GitHub Copilot to work on tasks"
[5]: https://github.blog/ai-and-ml/github-copilot/5-tips-for-writing-better-custom-instructions-for-copilot/?utm_source=chatgpt.com "5 tips for writing better custom instructions for Copilot"
[6]: https://stackoverflow.com/questions/76509513/how-to-use-github-copilot-for-multiple-files?utm_source=chatgpt.com "How to use GitHub Copilot for multiple files?"
