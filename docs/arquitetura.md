# Arquitetura — Multi-Agent Repository Analyzer

Documentação técnica de como o pipeline funciona internamente. Para a visão geral do produto, veja o
[README](../README.md).

## Visão em 3 passos

![Visão simplificada](../assets/visao-simplificada.png)

Entrada (repositório) → Análise (5 agentes + validação) → Saída (JSON/HTML/PDF).

## Pipeline completo

![Pipeline do Multi-Agent Repository Analyzer](../assets/pipeline-diagrama.png)

**Legenda de cores:** cinza = entrada · azul = Python determinístico · laranja = agente de IA ·
verde = arquivo de saída.

1. **`RepositoryProvider`** entrega o repositório como um diretório local — nem o inspector nem os
   agentes sabem de onde ele veio nem quais credenciais foram usadas para obtê-lo.
   - `LocalRepositoryProvider` — diretório já em disco, montado somente leitura.
   - `GitHubRepositoryProvider` — clone superficial (`depth=1`) em workspace temporário, removido ao
     final da análise (sucesso ou falha). Apenas HTTPS é aceito; autenticação via `GITHUB_TOKEN`
     passada como variável de ambiente de vida curta ao subprocesso de clone.

2. **`RepositoryInspector`** varre o repositório uma única vez, sem LLM, e monta o `RepositoryContext`
   compartilhado por todos os agentes: linguagens, frameworks, migrations, diretórios de
   models/controllers/routes/services/tests, Dockerfiles, arquivos de CI/CD, e caminhos — nunca
   conteúdo — de arquivos sensíveis (`.env`, `*.pem`, `id_rsa*`).

3. **5 agentes especializados** leem o mesmo contexto:

   ![Os 5 agentes especializados](../assets/agentes-grade.png)

   - **Architecture**, **Performance**, **Code Quality** e **Business Rules** usam o modo **legacy**:
     agente CrewAI com tool-calling autônomo e 3 ferramentas somente-leitura (`read_repository_file`,
     `list_repository_files`, `search_repository`).
   - **Security** tem dois modos, controlados por `SECURITY_ANALYSIS_MODE`:
     - `legacy` (padrão) — mesmo modelo dos demais agentes.
     - `two_pass` — pipeline determinístico: Pass A (planejamento de queries) → retrieval
       determinístico (sem tool-calling autônomo) → montagem de evidência contextual → Pass C
       (geração de achados ancorados em `evidence_refs`) → localização de `file`/`line`/`evidence` →
       `EvidenceValidator`. Falhas no two-pass **não** fazem fallback silencioso para legacy.
   - **Code Quality** também suporta `hybrid` (detectores estruturais determinísticos + agente de LLM,
     padrão) ou `structural` (só detectores, sem LLM).

4. **`EvidenceValidator`** reconfere mecanicamente o `file`/`line`/`evidence` de cada achado contra o
   repositório real, marcando `validated` / `unverified` / `invalid` — nunca reescreve o achado:

   ![Como uma evidência é validada](../assets/validacao-evidencia.png)

5. **Report Agent** só sintetiza achados já produzidos (resumo executivo, riscos, recomendações); não
   reanalisa o código. Apenas achados `validated` alimentam a síntese executiva; achados `unverified`
   aparecem separadamente em "Pontos para Investigação".

6. O **`Report`** final é montado deterministicamente em Python — uma recomendação só vira
   "prioridade" se estiver ancorada em evidência `validated`.

## Saída

Cada execução grava `report.json`, `report.html` e `report.pdf` em `reports/<analysis_id>/`. O
`report.json` também expõe, por agente: `analysis_mode_by_agent` (`legacy`/`two_pass`) e
`two_pass_by_agent` (telemetria por etapa: duração, outcome, contagem de achados).

## Reexecução parcial

`--report-only <report.json>` permite rodar apenas o Report Agent a partir de achados já salvos —
útil quando só a etapa de síntese falhou ou expirou, sem pagar de novo pelas chamadas de LLM dos 5
agentes especializados.

## Benchmark

O analyzer pode ser comparado a uma referência **parcial e curada** exportada de um scanner externo
(hoje: SonarQube em um projeto real). Isso não é ground truth absoluto — a referência lista casos
conhecidos para medir recall, não o universo de todos os achados possíveis. Cada finding da
referência carrega adjudicação humana (`confirmed` / `contextual` / `rejected`). O matching é
determinístico: domínio compatível + `file` exatamente igual + `line` dentro de uma tolerância — sem
embeddings, sem LLM-as-judge.

## Limitações conhecidas

- Apenas análise estática — nenhuma execução do código do repositório analisado.
- A confiabilidade da saída estruturada depende da LLM subjacente; modelos locais menores podem
  falhar em produzir saída interpretável ou usar tools de forma confiável em tarefas complexas.
- Detecção de framework/linguagem é heurística (convenções de nome de arquivo/caminho), não um parser completo.
- Os 5 agentes rodam sequencialmente hoje; a estrutura já é orientada a dados para permitir migrar
  para execução concorrente sem alterar a assinatura de chamada de nenhum agente.
- `REPORT_LANGUAGE` traduz apenas os rótulos fixos do template; o conteúdo gerado pela LLM sai no
  idioma em que o modelo responder.

Fontes vetoriais dos diagramas (zoom sem perda) estão no repositório original, em `docs/*.svg`.
