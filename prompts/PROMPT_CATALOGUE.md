# Prompt Log Catalog

## Metadata

- **Target Model:** claude-sonnet-4-6 (via Abacus AI) / fallback: gpt-4o, claude-3-5-sonnet
- **Catalog Version:** 1.1
- **Last Updated:** 2026-06-30
- **Owner:** <anonimizado>

---

## Record #001

### Identification

- **ID:** PROMPT-001
- **Name:** Onboarding RAG Assistant — Main Chat
- **Version:** 1.2
- **Owner:** <anonimizado>
- **Date:** 2026-05-15

### Objective

> Answer natural language questions about an indexed codebase. This is the core prompt of the application — used for every message sent by the user in the Chat. It solves the onboarding problem by allowing the new dev to ask questions like "What does the authentication module do?" and receive contextualized answers grounded in the repository's real code.

### Context of Use

> Invoked by `LlmClient.generate_answer()` in `backend/app/infrastructure/llm_client.py` on every call to the `POST /api/chat/{repo_id}` endpoint. Frequency: every user question in the chat. Typical latency: 3–8s.

### Prompt Template


```
[SYSTEM]
Você é um assistente especializado em onboarding de desenvolvedores em codebases.
Sua função é ajudar novos desenvolvedores a entender código existente, explicando:

* O que o código faz e por que foi implementado dessa forma
* Como diferentes partes se relacionam
* Conceitos e padrões utilizados

Baseie suas respostas SOMENTE no contexto fornecido. Se o contexto não for suficiente,
diga isso claramente. Use linguagem clara e exemplos quando relevante.

[USER]
Com base no código recuperado da codebase, responda:

{question}

Contexto da codebase:
{context_chunks}

Forneça uma resposta clara e objetiva.
```

### Parameters

| Parameter | Type | Description | Example |
| :--- | :--- | :--- | :--- |
| `question` | string | User's natural language question | `"O que faz o módulo de autenticação?"` |
| `context_chunks` | list[dict] | List of up to 5 code snippets retrieved via RAG, each with `text`, `file_path`, and `start_line` | `[{"text": "def signup(...):", "metadata": {"file_path": "auth_service.py", "start_line": 12}}]` |

### Execution Example

**Input:**


```
question: "Como funciona o sistema de embeddings? Qual modelo é usado?"

context_chunks:
[Fonte 1] backend/app/services/embedding_service.py (linha 45):
```
class EmbeddingService:
    def embed_texts(self, texts: list[str]) -> list[list[float]]:
        if self._provider == "openai":
            return self._embed_openai(texts)
        return self._embed_local(texts)

```
[Fonte 2] backend/app/infrastructure/settings.py (linha 28):
```
    embedding_provider: str = Field(default="local")
    embedding_model: str = Field(default="all-MiniLM-L6-v2")
    embedding_dim: int = Field(default=384)

```

```

**Output obtained:**


```
O sistema de embeddings do [ANONYMIZED_TOOL] suporta dois providers configuráveis via variável de ambiente `EMBEDDING_PROVIDER`:

1. **Local** (padrão): usa o modelo `all-MiniLM-L6-v2` da biblioteca `sentence-transformers`, produzindo vetores de 384 dimensões. Não requer chave de API.
2. **OpenAI**: usa o modelo `text-embedding-3-small`, produzindo vetores de 1536 dimensões. Requer `OPENAI_API_KEY`.

A troca entre providers é feita pela classe `EmbeddingService` que delega para `_embed_openai()` ou `_embed_local()` conforme a configuração.
```

### Quality Evaluation

- **Estimated Success Rate:** 78% (questions evaluated as "useful" in manual tests with 20 questions about [ANONYMIZED_TOOL] itself)
- **Failure Cases:**
    - Questions about business logic not present in the code (e.g., "why was PostgreSQL chosen?")
    - Questions about non-indexed code (files outside the 15 supported types)
    - Highly abstract questions without reference to a specific module
- **Mitigation Strategy:** Explicit instruction in the SYSTEM to declare insufficient context instead of hallucinating; limiting to 5 chunks per call; template-based fallback when LLM_API_KEY is not configured.

### Version History

| Version | Change | Reason |
| :--- | :--- | :--- |
| 1.0 | Initial version in English | Baseline implementation |
| 1.1 | Translated to Portuguese, added explanatory bullets in SYSTEM | Improve response quality for Brazilian devs |
| 1.2 | Instruction "Baseie suas respostas SOMENTE no contexto fornecido" | Reduce hallucinations observed in testing |

---

## Record #002

### Identification

- **ID:** PROMPT-002
- **Name:** "Why?" Explanation — Decisions via Commit History
- **Version:** 1.0
- **Owner:** <anonimizado>
- **Date:** 2026-05-20

### Objective

> Answer questions about the *historical decisions* of a module by analyzing classified commits. Solves the classic onboarding question: "Why was this code written this way?" or "Why were there so many changes in this module?". The AI synthesizes evidence from real commits into a coherent narrative.

### Context of Use

> Invoked by `WhyExplanationService._synthesize_explanation()` in `backend/app/services/timeline_service.py` when the user accesses the `GET /api/history/{repo_id}/why?module=&question=` endpoint. Frequency: on-demand, typically 1–3x per onboarding session.

### Prompt Template


```
[SYSTEM]
(herda PROMPT-001 — mesmo SYSTEM_PROMPT do LlmClient)

[USER]
Based on the commit history of module '{module_path}', answer: {question}

Evidence from commits:
[Fonte 1] {commit_timestamp} ({commit_category}) {commit_summary}
[Fonte 2] {commit_timestamp} ({commit_category}) {commit_summary}
...
```

### Parameters

| Parameter | Type | Description | Example |
| :--- | :--- | :--- | :--- |
| `module_path` | string | Relative path of the analyzed module | `"backend/app/services/auth_service.py"` |
| `question` | string | Question about module decisions | `"Por que esse arquivo foi refatorado tanto?"` |
| `commit_timestamp` | string | Commit date/time (ISO 8601) | `"2026-04-15T14:32:00"` |
| `commit_category` | string | Classified commit category | `"refactor"`, `"bugfix"`, `"feature"` |
| `commit_summary` | string | Summarized commit message | `"Extrair AuthService para remover duplicação"` |

### Execution Example

**Input:**


```
module_path: "backend/app/services/embedding_service.py"
question: "Por que houve tantas mudanças nesse módulo?"

Evidence:
[2026-04-10] (feature) Add OpenAI embedding provider support
[2026-04-12] (performance) Add ThreadPoolExecutor for parallel embedding batches
[2026-04-14] (refactor) Extract EmbeddingService from ingestion pipeline
[2026-04-18] (bugfix) Fix dimension mismatch between local and openai embeddings

```

**Output obtained:**


```
O módulo `embedding_service.py` passou por intensa evolução porque é um componente central
com múltiplos fornecedores. Inicialmente suportava apenas embeddings locais; depois foi
necessário adicionar suporte ao provider OpenAI (feature), otimizar com paralelismo
via ThreadPoolExecutor para reduzir o tempo de indexação de ~220s para ~12s (performance),
e corrigir uma incompatibilidade de dimensões entre os dois providers (bugfix).
```

### Quality Evaluation

- **Estimated Success Rate:** 72% (coherent syntheses matching actual history)
- **Failure Cases:** Repositories with poor commit messages (e.g., "fix", "update") — the AI lacks sufficient context.
- **Mitigation Strategy:** Classification by keyword matching before calling the LLM; detailed template-based fallback when LLM is unavailable.

---

## Record #003

### Identification

- **ID:** PROMPT-003
- **Name:** Guided Tour Generation — Module Walkthrough
- **Version:** 1.1
- **Owner:** <anonimizado>
- **Date:** 2026-05-18

### Objective

> Generate an explanatory walkthrough for the most critical modules of a codebase, automatically identified by a score of complexity × churn × coupling. The new dev receives a step-by-step "tour" through the modules they most need to understand, without needing to figure out where to start.

### Context of Use

> Invoked by `TourGenerationService` in `backend/app/services/tour_service.py` at the `POST /api/tours/{repo_id}/generate` endpoint. Executed once per repository; result stored in PostgreSQL. Can be manually regenerated.

### Prompt Template


```
[SYSTEM]
Você é um especialista em engenharia de software encarregado de criar material de
onboarding para novos desenvolvedores. Seu objetivo é explicar módulos de código
de forma clara, progressiva e acessível para alguém que nunca viu esse repositório.

[USER]
Crie um walkthrough de onboarding para o módulo '{module_name}' do repositório '{repo_name}'.

Informações do módulo:

* Score de importância: {importance_score:.2f} (0-1, quanto maior mais crítico)
* Complexidade ciclomática média: {avg_complexity:.1f}
* Frequência de mudança (commits/arquivo): {avg_churn:.1f}
* Acoplamento (imports/arquivo): {avg_coupling:.1f}

Arquivos principais do módulo:
{file_list}

Trecho de código representativo:
{code_sample}

Gere um walkthrough com:

1. O que este módulo faz (2-3 frases)
2. Por que é importante para o projeto
3. Conceitos-chave que o dev precisa entender
4. Pontos de atenção ou complexidades
5. Por onde começar a explorar
```

### Parameters

| Parameter | Type | Description | Example |
| :--- | :--- | :--- | :--- |
| `module_name` | string | Name/path of the module | `"app/services"` |
| `repo_name` | string | Repository name | `"[ANONYMIZED_TOOL]"` |
| `importance_score` | float | Ranking score 0-1 | `0.73` |
| `avg_complexity` | float | Average cyclomatic complexity | `4.2` |
| `avg_churn` | float | Average commits per file | `8.5` |
| `avg_coupling` | float | Average imports per file | `6.1` |
| `file_list` | string | List of module files | `"embedding_service.py\nchat_service.py"` |
| `code_sample` | string | Representative snippet (up to 800 chars) | `"class EmbeddingService:\n    def embed_texts..."` |

### Execution Example

**Input:** module `app/services` of repository `[ANONYMIZED_TOOL]`, score 0.71

**Output obtained:**


```
## Módulo: app/services

**O que faz:** Este módulo concentra toda a lógica de negócio do [ANONYMIZED_TOOL]. É onde
o RAG pipeline é orquestrado, os embeddings são gerados, os commits são classificados
e os tours são produzidos.

**Por que é importante:** É o coração da aplicação — todas as chamadas dos controllers
passam por aqui antes de chegar à infraestrutura. Entender este módulo é entender 80%
do que o [ANONYMIZED_TOOL] faz.

**Conceitos-chave:**

* RAG (Retrieval-Augmented Generation): busca semântica + LLM
* Arquitetura Hexagonal: os services dependem de ports (interfaces), não de implementações
* Pipeline de indexação: Git clone → chunking → embedding → ChromaDB

**Pontos de atenção:** O `embedding_service.py` tem dois providers (local e OpenAI)
com comportamentos diferentes de latência. O `tour_service.py` depende do sistema
de arquivos estar disponível.

**Por onde começar:** `chat_service.py` → `retrieval_service.py` → `embedding_service.py`
```

### Quality Evaluation

- **Estimated Success Rate:** 85% (walkthroughs evaluated as "useful" by the team)
- **Failure Cases:** Modules with very small or generic files (e.g., pure `__init__.py`) — too little context to generate an explanation.
- **Mitigation Strategy:** Filter modules with fewer than 50 total lines; pre-select the most representative code snippet (largest function, not the shortest).

---

## Record #004

### Identification

- **ID:** PROMPT-004
- **Name:** Onboarding Quality Report
- **Version:** 1.0
- **Owner:** <anonimizado>
- **Date:** 2026-05-25

### Objective

> Generate an interpreted analysis of collected onboarding metrics (response latency, session completion rate, feedback usefulness/correctness score). Transforms raw data into actionable recommendations for the tech lead aiming to improve the team's onboarding process.

### Context of Use

> Invoked by `ReportingService.build_quality_report()` at the `GET /api/metrics/{repo_id}/report` endpoint. Frequency: on-demand by the admin/tech lead. Rarely called more than 1x per week.

### Prompt Template


```
[SYSTEM]
Você é um especialista em métricas de engenharia de software e melhoria de processos.
Analise as métricas de onboarding fornecidas e gere recomendações práticas e específicas.

[USER]
Analise as métricas de qualidade de onboarding do repositório para o período {period_start} a {period_end}:

Métricas coletadas:

* Total de sessões: {total_sessions}
* Taxa de conclusão do fluxo de onboarding: {completion_rate:.1%}
* Latência P50 de resposta: {p50_latency:.1f}s
* Latência P95 de resposta: {p95_latency:.1f}s
* Nota média de utilidade das respostas: {usefulness:.2f}/5.0
* Nota média de correção das respostas: {correctness:.2f}/5.0
* Cobertura de feedback: {feedback_coverage:.1%}

Classificação geral: {quality_label} ({overall_quality:.1%})

Gere:

1. Diagnóstico em 2-3 frases
2. Principais pontos de atenção
3. 3 recomendações concretas e priorizadas
```

### Parameters

| Parameter | Type | Description | Example |
| :--- | :--- | :--- | :--- |
| `period_start` | string | Start of the analyzed period | `"2026-05-01"` |
| `period_end` | string | End of the analyzed period | `"2026-05-31"` |
| `total_sessions` | int | Total onboarding sessions | `47` |
| `completion_rate` | float | Session completion rate | `0.68` |
| `p50_latency` | float | Median latency in seconds | `4.2` |
| `p95_latency` | float | P95 latency in seconds | `12.8` |
| `usefulness` | float | Average usefulness score (1-5) | `3.8` |
| `correctness` | float | Average correctness score (1-5) | `3.6` |
| `feedback_coverage` | float | % of responses with feedback | `0.42` |
| `quality_label` | string | Calculated quality label | `"good"` |

### Execution Example

**Input:** 47 sessions, completion 68%, P50 4.2s, usefulness 3.8/5, correctness 3.6/5

**Output obtained:**


```
**Diagnóstico:** O onboarding está funcionando de forma satisfatória com taxa de conclusão
de 68% e notas de utilidade acima da média. O principal gargalo é a latência P95 de 12.8s
que pode causar abandono em perguntas complexas.

**Pontos de atenção:**

* Latência P95 elevada (12.8s) — risco de frustração para 5% das perguntas
* Cobertura de feedback de apenas 42% — amostra insuficiente para conclusões robustas

**Recomendações:**

1. Implementar streaming de resposta para reduzir percepção de latência (impacto: alto)
2. Adicionar prompt de feedback ao final de cada resposta para aumentar cobertura acima de 70%
3. Investigar as perguntas com maior latência — provavelmente chunks muito grandes sendo recuperados
```

### Quality Evaluation

- **Estimated Success Rate:** 80%
- **Failure Cases:** Periods with very little data (< 10 sessions) result in overly generic recommendations.
- **Mitigation Strategy:** Return a warning message when `total_sessions < 10`; this prompt is only invoked when there is sufficient data.

---

## Record #005

### Identification

- **ID:** PROMPT-005
- **Name:** Commit Category Classification
- **Version:** 2.0
- **Owner:** <anonimizado>
- **Date:** 2026-05-12

### Objective

> Automatically classify commit messages into semantic categories (bugfix, feature, refactor, performance, documentation, test, infrastructure, dependency, style, other). Feeds the decision timeline and the "Why?" endpoint. Critical for the quality of historical explanations.

### Context of Use

> Invoked by `DecisionClassificationService.classify_batch()` in `backend/app/services/commit_history_service.py` during the history indexing phase. In the current implementation, it uses keyword classification (regex) before falling back to the LLM. Called once per repository, in batches of up to 200 commits.

### Prompt Template

> **Note:** The current version (2.0) uses hybrid classification — it first tries keyword patterns; the LLM is called only when the pattern matching confidence score falls below 0.6. Version 1.0 called the LLM for all commits.


```
[SYSTEM]
Você é um especialista em engenharia de software que analisa históricos de commits.
Classifique cada commit em exatamente UMA das categorias abaixo.
Responda SOMENTE com o JSON solicitado, sem texto adicional.

Categorias válidas: bugfix, feature, refactor, performance, documentation,
test, infrastructure, dependency, style, other

[USER]
Classifique os seguintes commits. Para cada um, retorne:
{"commit_id": "...", "category": "...", "confidence": 0.0-1.0, "summary": "resumo em português em 1 frase"}

Commits:
{commits_batch}
```

### Parameters

| Parameter | Type | Description | Example |
| :--- | :--- | :--- | :--- |
| `commits_batch` | string | Formatted list of commits (id + message + touched files) | `"abc123: 'fix null pointer in auth' [auth_service.py]"` |

### Execution Example

**Input:**


```
abc123: "fix null pointer in auth service when token expires" [auth_service.py, dependencies.py]
def456: "add OpenAI embedding provider with parallel batch processing" [embedding_service.py, settings.py]
ghi789: "refactor ingestion pipeline to use hexagonal architecture" [ingestion_service.py, ports.py, main.py]

```

**Output obtained:**

```json
[
    {
        "commit_id": "abc123",
        "category": "bugfix",
        "confidence": 0.95,
        "summary": "Corrige null pointer em auth_service quando token expira"
    },
    {
        "commit_id": "def456",
        "category": "feature",
        "confidence": 0.92,
        "summary": "Adiciona suporte a embeddings OpenAI com processamento paralelo em batch"
    },
    {
        "commit_id": "ghi789",
        "category": "refactor",
        "confidence": 0.88,
        "summary": "Refatora pipeline de indexação para arquitetura hexagonal"
    }
]

```

### Quality Evaluation

* **Estimated Success Rate:** 91% (manual validation on 50 commits from [ANONYMIZED_TOOL] itself)
* **Failure Cases:** Commits with highly generic messages (`"wip"`, `"update"`, `"fix"`) are classified as `other` with low confidence.
* **Mitigation Strategy:**
* Version 2.0 uses keyword matching first (rules in `_CATEGORY_PATTERNS`) — avoids unnecessary LLM calls.
* LLM is invoked only for commits where keyword match confidence < 0.6.
* `other` is a valid category for ambiguous commits — prevents forced misclassification.



---

## Record #006

### Identification

* **ID:** PROMPT-006
* **Name:** Dependency Graph Analysis — Module Explanation
* **Version:** 1.0
* **Owner:** 
* **Date:** 2026-05-22

### Objective

> Generate a natural language explanation of a module's role within the codebase's dependency graph. The user sees the import graph on the frontend and can click on a node to get an explanation of "what is this module and why do so many others depend on it?".

### Context of Use

> Invoked by `DependencyGraphService` at the `GET /api/graph/{repo_id}/module/{module_path}` endpoint. Frequency: on-demand when clicking a graph node. Typically 3–10 calls per graph exploration session.

### Prompt Template

```
[SYSTEM]
Você é um especialista em arquitetura de software que ajuda desenvolvedores a entender
a estrutura de uma codebase por meio de seu grafo de dependências.

[USER]
Explique o papel do módulo '{module_path}' na codebase '{repo_name}'.

Dados do grafo de dependências:
- Módulos que DEPENDEM deste: {dependents} ({in_degree} dependentes)
- Módulos dos quais este DEPENDE: {dependencies} ({out_degree} dependências)
- Centralidade no grafo (0-1): {centrality:.2f}
- Tipo de nó: {node_type}

Trecho de código do módulo:
{code_preview}

Explique em 3-5 frases:
1. O que este módulo faz
2. Por que outros módulos dependem dele (se alta centralidade)
3. Recomendação para o dev novato (deve entender antes ou depois de outros módulos?)
```

### Parameters

| Parameter | Type | Description | Example |
| --- | --- | --- | --- |
| `module_path` | string | Module path in the repo | `"app/ports.py"` |
| `repo_name` | string | Repository name | `"[ANONYMIZED_TOOL]"` |
| `dependents` | string | List of dependent modules | `"chat_service, tour_service, ingestion_service"` |
| `in_degree` | int | Number of modules depending on this one | `8` |
| `dependencies` | string | Modules this one depends on | `"(nenhum)"` |
| `out_degree` | int | Number of dependencies for this module | `0` |
| `centrality` | float | Graph centrality (betweenness) | `0.82` |
| `node_type` | string | Type: `"hub"`, `"leaf"`, `"bridge"`, `"isolated"` | `"hub"` |
| `code_preview` | string | First 500 chars of the file | `"class LLMPort(Protocol):..."` |

### Execution Example

**Input:** module `app/ports.py`, 8 dependents, 0 dependencies, centrality 0.82

**Output obtained:**

```
O módulo `ports.py` define as interfaces (Protocols do Python) que desacoplam a lógica
de negócio das implementações concretas — é o coração da arquitetura hexagonal do projeto.

Oito módulos dependem dele porque todos os services e controllers acessam infraestrutura
(banco de dados, ChromaDB, LLM) exclusivamente através dessas interfaces, garantindo
testabilidade e substituibilidade de implementações.

**Para o dev novato:** entenda este módulo PRIMEIRO, antes de qualquer outro. Lendo
`ports.py` você terá uma visão completa das capacidades do sistema sem precisar entender
as implementações concretas.
```

### Quality Evaluation

* **Estimated Success Rate:** 88%
* **Failure Cases:** Modules of `isolated` type (no dependents or dependencies) tend to generate vague explanations.
* **Mitigation Strategy:** For isolated modules, skip the LLM call and return a default explanation based on the filename.

---

## Prompt Summary

| ID | Name | Endpoint | Usage Freq. | Version |
| --- | --- | --- | --- | --- |
| PROMPT-001 | Main RAG Chat | `POST /api/chat/{repo_id}` | High (every question) | 1.2 |
| PROMPT-002 | "Why?" Explanation | `GET /api/history/{repo_id}/why` | Medium (on-demand) | 1.0 |
| PROMPT-003 | Guided Tour — Walkthrough | `POST /api/tours/{repo_id}/generate` | Low (1x per repo) | 1.1 |
| PROMPT-004 | Quality Report | `GET /api/metrics/{repo_id}/report` | Low (weekly) | 1.0 |
| PROMPT-005 | Commit Classification | Internal — indexing pipeline | Low (1x per repo) | 2.0 |
| PROMPT-006 | Graph Node Analysis | `GET /api/graph/{repo_id}/module/{path}` | Medium (per click) | 1.0 |
| PROMPT-007 | Branch Analysis | `POST /api/repos/{id}/analyze-branch` | Medium (on-demand) | 1.0 |
| PROMPT-008 | Documentation Generator | `POST /api/repos/{id}/generate-doc` | Low (on-demand) | 1.0 |
| PROMPT-009 | Drift Interpretation | `POST /api/repos/{id}/graph/diff/interpret` | Low (on-demand) | 1.0 |
| PROMPT-010 | Tech Debt Analysis | `POST /api/repos/{id}/tech-debt/analyse` | Low (on-demand) | 1.0 |

---

## Record #007

### Identification

* **ID:** PROMPT-007
* **Name:** Branch Analysis with Risk Assessment
* **Version:** 1.0
* **Owner:** 
* **Date:** 2026-06-05

### Objective

> Analyze the changes of a feature branch against the base, calculate a risk score based on the altered files (complexity + coupling), and generate a natural language summary highlighting the main risks and impacts. Helps devs review what changed before merging.

### Context of Use

> Invoked by `BranchAnalysisService.analyze()` at the `POST /api/repos/{id}/analyze-branch` endpoint. Receives `branch_name` and `base_branch`. Frequency: on-demand, typically before a PR review.

### Prompt Template

```
[SYSTEM]
Você é um engenheiro de software sênior experiente em code review e análise de risco.
Analise as mudanças de branch e forneça insights acionáveis e objetivos.
Responda em português.

[USER]
Analise as mudanças da branch '{branch}' em relação a '{base_branch}' no repositório '{repo_name}'.

Arquivos alterados ({files_changed} total):
{changed_files_list}

Risk score calculado: {risk_score:.0f}/100
(baseado em complexidade ciclomática média e acoplamento dos arquivos alterados)

Forneça:
1. Resumo das mudanças em 2-3 frases
2. Principais riscos identificados
3. Módulos mais críticos tocados
4. Recomendação de merge (safe/review/caution)
```

### Parameters

| Parameter | Type | Description | Example |
| --- | --- | --- | --- |
| `branch` | string | Feature branch name | `"feature/add-webhook-support"` |
| `base_branch` | string | Base branch for comparison | `"main"` |
| `repo_name` | string | Repository name | `"[ANONYMIZED_TOOL]"` |
| `files_changed` | int | Total changed files | `12` |
| `changed_files_list` | string | List of files with metrics | `"main.py (complexity: 8, in: 5)"` |
| `risk_score` | float | Calculated score 0-100 | `67.0` |

### Quality Evaluation

* **Estimated Success Rate:** 83%
* **Failure Cases:** Branches with only docs/tests changes — low risk score but LLM might overestimate risk.
* **Mitigation Strategy:** Filter `.md` and `test_*` files from the risk score before calling the LLM.

---

## Record #008

### Identification

* **ID:** PROMPT-008
* **Name:** Module Documentation Generator
* **Version:** 1.0
* **Owner:** 
* **Date:** 2026-06-05

### Objective

> Generate automatic documentation (in README.md format) for a specific module, combining static analysis of indexed code with the commit history. Useful for projects with missing or outdated docs.

### Context of Use

> Invoked by `DocGeneratorService.generate()` at the `POST /api/repos/{id}/generate-doc` endpoint. Receives `module_path`. Frequency: on-demand by the dev or tech lead.

### Prompt Template

```
[SYSTEM]
Você é um escritor técnico especializado em documentação de software.
Gere documentação clara, precisa e útil para desenvolvedores.
Responda em português brasileiro com formatação Markdown.

[USER]
Gere um README.md detalhado para o módulo '{module_path}' do repositório '{repo_name}'.

Código-fonte (chunks mais relevantes):
{code_chunks}

Histórico recente de commits:
{recent_commits}

Métricas do módulo:
- Complexidade ciclomática média: {avg_complexity:.1f}
- Linhas de código: {loc}
- Linguagem: {language}

Inclua no README gerado:
1. Descrição e propósito do módulo
2. Responsabilidades principais
3. Como usar (exemplo de código se aplicável)
4. Dependências e integrações
5. Notas de manutenção (baseado no histórico de mudanças)
```

### Quality Evaluation

* **Estimated Success Rate:** 79%
* **Failure Cases:** Very small modules (< 30 lines) result in superficial documentation; generated code cannot be tested by the LLM.
* **Mitigation Strategy:** Limit invocation to modules with `loc >= 30`; include explicit instructions to base content solely on the provided code.

---

## Record #009

### Identification

* **ID:** PROMPT-009
* **Name:** Architectural Drift Interpretation
* **Version:** 1.0
* **Owner:** 
* **Date:** 2026-06-15

### Objective

> Interpret in natural language the result of comparing two dependency graph snapshots (drift report). Transforms structured data (added/removed nodes, added/removed edges, drift score) into a readable diagnosis about what changed architecturally and its implications.

### Context of Use

> Invoked by the `POST /api/repos/{id}/graph/diff/interpret` endpoint in `dependency_graph_controller.py`. Triggered when the user clicks "Interpret with AI" after viewing the diff of two snapshots in `DriftTab.tsx`. Frequency: on-demand, usually 1x per drift analysis session.

### Prompt Template

```
[SYSTEM]
Você é um arquiteto de software experiente especializado em análise de evolução de
sistemas. Analise as mudanças estruturais de dependências e forneça insights sobre
impactos e riscos. Responda SEMPRE em português brasileiro.

[USER]
Analise o drift arquitetural do repositório '{repo_name}' entre dois snapshots.

Resumo das mudanças estruturais:
- Drift score: {drift_score:.1f}% ({drift_label})
- Nós adicionados: {added_nodes} módulos novos
- Nós removidos: {removed_nodes} módulos deletados
- Arestas adicionadas: {added_edges} novas dependências
- Arestas removidas: {removed_edges} dependências removidas

Detalhes das mudanças:
{diff_details}

Forneça uma interpretação de 3-5 parágrafos cobrindo:
1. O que mudou estruturalmente e o nível de impacto
2. Riscos ou melhorias arquiteturais detectadas
3. Módulos que merecem atenção especial
4. Recomendação geral para o time
```

### Parameters

| Parameter | Type | Description | Example |
| --- | --- | --- | --- |
| `repo_name` | string | Repository name | `"[ANONYMIZED_TOOL]"` |
| `drift_score` | float | Percentage of altered elements | `23.4` |
| `drift_label` | string | Qualitative drift label | `"moderate"`, `"high"`, `"low"` |
| `added_nodes` | int | Number of new modules | `3` |
| `removed_nodes` | int | Number of removed modules | `1` |
| `added_edges` | int | Number of new dependencies | `7` |
| `removed_edges` | int | Number of removed dependencies | `2` |
| `diff_details` | string | List of altered elements | `"+ webhook_controller.py\n+ ..."` |

### Execution Example

**Input:** drift_score 18.5%, 2 added nodes, 0 removed, 5 added edges

**Output obtained:**

```
A evolução arquitetural registrada entre os dois snapshots indica uma expansão controlada
do sistema, com drift score de 18.5% classificado como moderado. Foram adicionados dois
novos módulos (webhook_controller e watchlist_controller) e cinco novas dependências,
sem remoção de módulos existentes — sinal de crescimento aditivo.

Os novos módulos se integram à camada de controllers com dependências para repositórios
de infraestrutura, seguindo o padrão hexagonal já estabelecido. As arestas adicionadas
são majoritariamente de controllers para adapters, o que é esperado e saudável.

Ponto de atenção: o webhook_controller passou a depender diretamente do ingestion_service,
criando acoplamento entre a camada de entrada externa e o pipeline de processamento.
Considerar se essa dependência deveria ser mediada por um port/interface.

Recomendação: o crescimento está dentro de parâmetros normais. Revisar o acoplamento
do webhook_controller antes do próximo sprint.
```

### Quality Evaluation

* **Estimated Success Rate:** 87%
* **Failure Cases:** Very low drift score (< 5%) makes interpretation trivial; very high drift score (> 50%) leads to superficial analysis due to overwhelming changes.
* **Mitigation Strategy:** For drift < 5%, return a default message without calling the LLM ("Minimal changes detected — no significant architectural impact"); for drift > 50%, include an explicit warning in the prompt regarding the massive change.

---

## Record #010

### Identification

* **ID:** PROMPT-010
* **Name:** Technical Debt Analysis by Code Best Practices
* **Version:** 1.0
* **Owner:** 
* **Date:** 2026-06-30

### Objective

> Evaluate a repository's technical debt by comparing critical files (top hotspots) against recognized industry principles: Clean Code, SOLID, DRY, KISS, YAGNI, and Clean Architecture. Transforms quantitative metrics (CC, churn, LOC, coupling) into a categorized qualitative diagnosis, providing prioritized actions for debt reduction.

### Context of Use

> Invoked by `TechDebtService._generate_llm_summary()` in `backend/app/services/tech_debt_service.py`, triggered by the `POST /api/repos/{id}/tech-debt/analyse` endpoint. Frequency: on-demand by the user when clicking "Analisar Agora" in the Tech Debt dashboard. It is not called during automatic indexing (which only uses `take_snapshot()` without LLM). Typical latency: 4–10s.

### Prompt Template

```
[SYSTEM]
Você é especialista em qualidade de software e arquitetura.
Analise dados de dívida técnica e responda em Português do Brasil de forma CONCISA.
Use Markdown simples: **negrito**, bullet com -, listas numeradas. Limite: ~450 tokens.

[USER]
Score médio de dívida: {avg_score:.1f}/100 | Tendência: {trend_label}
CC média: {avg_complexity:.1f} | Churn médio: {avg_churn:.1f} commits/arquivo |
LOC médio: {avg_loc:.0f} | Acoplamento: {coupling:.1f} imports/arquivo

Arquivos críticos analisados:
- {relative_path} | score: {hotspot_score:.0f} | CC: {complexity:.1f} | churn: {churn} commits | {loc} LOC
(... até 8 arquivos)

Avalie considerando: Clean Code, SOLID, DRY, KISS, YAGNI, Clean Architecture e Design Patterns.

Responda neste formato exato:
**Score de Dívida:** [0-100] - [2 frases de justificativa]

**Principais Problemas:**
- [Categoria SOLID/Clean Code/etc]: [descrição com arquivo de exemplo]
- [Categoria]: [descrição]
- [Categoria]: [descrição]

**Ações Priorizadas:**
1. [ação concreta] - Impacto: alto/médio/baixo
2. [ação concreta] - Impacto: alto/médio/baixo
3. [ação concreta] - Impacto: alto/médio/baixo

**Diagnóstico:** {trend_label} - [1 frase explicando o motivo]
```

### Parameters

| Parameter | Type | Description | Example |
| --- | --- | --- | --- |
| `avg_score` | float | Average debt score of the repo (0–100) | `62.3` |
| `trend_label` | string | Debt trend (`melhorando`/`estável`/`degradando`) | `"degradando"` |
| `avg_complexity` | float | Average cyclomatic complexity among hotspots | `8.4` |
| `avg_churn` | float | Average commits per file (last 6 months) | `14.2` |
| `avg_loc` | float | Average lines of code per file | `312` |
| `coupling` | float | Average imports per file | `9.1` |
| `files_lines` | string | Formatted list of top-8 hotspot files with metrics | `"- auth_service.py | score: 84 | CC: 12.3"` |

### Execution Example

**Input:** score 62.3, degradando, CC 8.4, churn 14.2, LOC 312, coupling 9.1

**Output obtained:**

```
**Score de Dívida:** 62/100 - O repositório apresenta dívida moderada-alta concentrada em
poucos arquivos críticos com alta complexidade ciclomática e churn elevado. A tendência de
degradação indica que as mudanças recentes aumentaram o débito sem refatoração compensatória.

**Principais Problemas:**
- Complexidade (Clean Code / SOLID SRP): auth_service.py com CC 12.3 concentra múltiplas
  responsabilidades — autenticação, sessão e hash de senha num único módulo.
- Acoplamento (DIP / Hexagonal): ingestion_service.py importa 11 módulos internos
  diretamente, violando o princípio de inversão de dependência.
- Churn (YAGNI): 3 arquivos com churn > 20 commits/6m sem redução de complexidade —
  indicativo de patches contínuos sem refatoração estrutural.

**Ações Priorizadas:**
1. Extrair PasswordHasher e SessionManager de auth_service.py (SRP) - Impacto: alto
2. Introduzir interface para ingestion pipeline e inverter dependência - Impacto: alto
3. Estabelecer feature freeze nos arquivos de alto churn para refatoração - Impacto: médio

**Diagnóstico:** degradando - O aumento de churn sem redução proporcional de complexidade
indica ciclos de correção acelerada (firefighting) em vez de desenvolvimento sustentável.
```

### Quality Evaluation

* **Estimated Success Rate:** 82%
* **Failure Cases:** Repositories with few files (< 5 hotspots) result in analysis with insufficient context; zeroed metrics (repo with no commits) cause the LLM to produce a generic analysis.
* **Mitigation Strategy:** Check `len(hotspots) >= 3` before calling the LLM; for repos with no commits (churn = 0 across all), return `llm_summary = ""` and display a message guiding the user to execute `git log`.

### Version History

| Version | Change | Reason |
| --- | --- | --- |
| 1.0 | Initial version | Implementation of endpoint `POST /tech-debt/analyse` (v2) |

```

```
