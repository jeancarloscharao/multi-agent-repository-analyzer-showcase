# Arquitetura do Multi-Agent Repository Analyzer

Como o pipeline funciona por dentro. Para a visão geral do produto, veja o [README](../README.md).

## Pipeline conceitual (estado atual)

```
RepositoryProvider → Inspector
  → agentes por modo (legacy / two_pass / structural / hybrid / off)
  → EvidenceValidator (+ checagem semântica no Security two-pass)
  → risco residual documentado (Security, determinístico)
  → sanitizador de saída (privacy)
  → Report Agent (síntese)
  → Report público → JSON / HTML / PDF
```

**Security `two_pass`:** Pass A planeja as queries, um retrieval determinístico busca a evidência,
uma etapa de política confere cobertura, Pass C gera achados ancorados nessa evidência, depois vêm
localização, `EvidenceValidator`, checagem semântica de domínio, o extrator de risco residual
documentado e, por fim, o merge com descarte de conflito.

**Architecture `two_pass`:** roda o mesmo pipeline compartilhado, mas ainda sem a checagem semântica
de domínio. Achados fisicamente corretos ficam em `unverified` e não entram como prioridade sozinhos.

**Performance e Qualidade de Código `structural`:** detectores PHP determinísticos, sem LLM
nenhuma envolvida.

## Visão em 3 passos

![Visão simplificada](../assets/visao-simplificada.png)

Entrada (repositório) → Análise (5 agentes + validação) → Saída (JSON/HTML/PDF).

## Pipeline completo

![Pipeline do Multi-Agent Repository Analyzer](../assets/pipeline-diagrama.png)

Legenda de cores: cinza é entrada, azul é Python determinístico, laranja é agente de IA, verde é
arquivo de saída.

1. O `RepositoryProvider` entrega o repositório como um diretório local. Nem o inspector nem os
   agentes sabem de onde ele veio nem quais credenciais foram usadas para obtê-lo.
   - `LocalRepositoryProvider`: diretório já em disco, montado somente leitura.
   - `GitHubRepositoryProvider`: clone superficial (`depth=1`) num workspace temporário, removido ao
     final da análise, com sucesso ou falha. Só HTTPS é aceito; a autenticação via `GITHUB_TOKEN` é
     passada como variável de ambiente de curta duração ao subprocesso de clone.

2. O `RepositoryInspector` varre o repositório uma única vez, sem LLM, e monta o `RepositoryContext`
   que todos os agentes vão compartilhar: linguagens, frameworks, migrations, diretórios de
   models/controllers/routes/services/tests, Dockerfiles, arquivos de CI/CD e os caminhos (nunca o
   conteúdo) de arquivos sensíveis como `.env`, `*.pem`, `id_rsa*`.

3. Os 5 agentes especializados leem esse mesmo contexto, cada um num modo configurável por domínio:

   ![Os 5 agentes especializados](../assets/agentes-grade.png)

   | Agente | Modos | Notas |
   |---|---|---|
   | Security | `legacy` / `two_pass` | `two_pass`: Pass A → retrieval → cobertura da política → Pass C → localização → `EvidenceValidator` → checagem semântica → risco residual documentado |
   | Architecture | `legacy` / `two_pass` | mesmo pipeline `two_pass` compartilhado, sem checagem semântica ainda; achados ficam `unverified`, não actionable |
   | Performance | `legacy` / `hybrid` / `structural` | `structural`: detectores PHP determinísticos (consulta em loop, `all()` sem paginação) |
   | Qualidade de Código | `legacy` / `hybrid` / `structural` | `structural`: função longa, classe grande, retornos excessivos, complexidade cognitiva |
   | Regras de Negócio | `legacy` / `off` | achados são hipóteses; `off` não executa o agente, o que é diferente de "rodou e achou zero" |

   O modo `legacy` é o mesmo modelo em todo agente: um agente CrewAI com tool-calling autônomo e 3
   ferramentas somente-leitura (`read_repository_file`, `list_repository_files`,
   `search_repository`). Nos modos `two_pass` e `structural`, se algo falhar no meio do caminho, não
   há fallback silencioso para o legacy: o erro fica registrado.

4. O `EvidenceValidator` reconfere mecanicamente o `file`/`line`/`evidence` de cada achado contra o
   repositório real, marcando `validated`, `unverified` ou `invalid`, sem nunca reescrever o achado.
   Nos domínios Security e Architecture em modo `two_pass`, um achado `validated` ainda passa por uma
   checagem semântica de domínio; só em Security essa checagem participa da decisão de virar
   prioridade:

   ![Como uma evidência é validada](../assets/validacao-evidencia.png)

5. Em Security, um extrator determinístico varre a política de segurança do próprio projeto em busca
   de risco residual aceito ou conhecido. Quando encontra, gera um achado `documented_residual_risk`,
   sempre `unverified` e nunca actionable, porque documentação é evidência secundária.

6. Um sanitizador de saída (`privacy`) redige segredos que tenham vazado para dentro de um achado
   (chaves de API, tokens do GitHub, JWTs, blocos de chave privada) antes de qualquer artefato ser
   persistido. Não existe `report-raw.json`: só versões já sanitizadas chegam ao disco.

7. O Report Agent só sintetiza achados que já foram produzidos, validados e sanitizados (resumo
   executivo, riscos, recomendações); ele não reanalisa o código. Só achados `validated` em contexto
   de produção ou config entram na síntese executiva. Os `unverified` aparecem à parte, em "Pontos
   para Investigação".

8. O `Report` final é montado deterministicamente em Python. Uma recomendação só vira "prioridade" se
   estiver ancorada em evidência `validated` em produção/config.

## Saída

Cada execução grava `report.json`, `report.html` e `report.pdf` em `reports/<analysis_id>/`, sempre
já sanitizados. O `report.json` também guarda, por agente, `analysis_mode_by_agent` e a telemetria de
cada etapa do `two_pass` (duração, outcome, contagem de achados).

## Reexecução parcial

`--report-only <report.json>` roda só o Report Agent a partir de achados já salvos. Serve para os
casos em que apenas a síntese falhou ou expirou, sem precisar pagar de novo pelas chamadas de LLM dos
5 agentes especializados.

## Perfil Cliente

`ANALYSIS_PROFILE=client` é um preset fixo para análises em repositórios confidenciais: LLM local via
Ollama (bloqueia provedores externos), `pt-BR`, Security/Architecture em `two_pass`, Performance e
Qualidade em `structural`, Regras de Negócio desligado. Um override que tente enfraquecer essa
configuração falha antes da análise começar, sem correção silenciosa. Se o repositório vier de um
clone GitHub, o clone participa da obtenção do código; a garantia é que o conteúdo analisado não é
enviado a provedores de LLM externos, não que "nada sai da máquina".

## Benchmark

O analyzer pode ser comparado a dois tipos de referência. Uma referência parcial e curada, exportada
de um scanner externo (hoje, SonarQube num projeto real), não é ground truth absoluto: lista casos
conhecidos para medir cobertura e recall confirmado, sem calcular precision, já que achados fora da
lista não viram falso positivo automaticamente. Uma referência fechada, onde todo o universo de casos
positivos e negativos foi adjudicado, permite calcular precision, recall e F1 de verdade: hoje o
benchmark fechado de Performance estrutural roda com 5 verdadeiros positivos, 0 falsos positivos, 0
falsos negativos e 12 verdadeiros negativos. Em ambos os casos, o matching é determinístico (domínio
compatível, `file` exatamente igual, `line` dentro de uma tolerância), sem embeddings e sem
LLM-as-judge.

## Limitações conhecidas

- Só análise estática. Nenhuma execução do código do repositório analisado.
- A confiabilidade da saída depende da LLM usada; modelos locais menores às vezes falham em produzir
  saída interpretável ou não usam as tools de forma confiável em tarefas mais complexas.
- Architecture ainda não tem checagem semântica de domínio; achados dessa categoria ficam
  `unverified` e não entram como prioridade sozinhos.
- Regras de Negócio continua desligado no Perfil Cliente; quando ligado, os achados são hipóteses.
- Detecção de framework e linguagem é heurística (convenções de nome de arquivo/caminho), não um
  parser completo.
- Os 5 agentes ainda rodam sequencialmente, mas o código já é orientado a dados o suficiente para
  migrar para execução concorrente sem mexer na assinatura de chamada de nenhum agente.
- `REPORT_LANGUAGE` traduz só os rótulos fixos do template. O conteúdo gerado pela LLM sai no idioma
  em que o modelo decidir responder.
- Relatórios exigem revisão humana antes de qualquer entrega externa; esta é uma versão de piloto
  assistido, não de análise autônoma.
