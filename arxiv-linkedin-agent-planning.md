# ArxivAgent: Sistema Multi-Agente para Curadoria e Disseminação de Papers de IA

## Visão Geral do Projeto

O ArxivAgent é um sistema autônomo multi-agente construído inteiramente em Rust que automatiza o ciclo completo de descoberta, análise e publicação de conteúdo sobre papers de IA no LinkedIn. O sistema opera como uma entidade cognitiva distribuída em agentes especializados, coordenados por algoritmos de consenso entre dois LLMs (Groq e OpenRouter), garantindo qualidade de análise enquanto respeita limites gratuitos de API.

A arquitetura foi projetada para execução serverless via GitHub Actions, tratando o repositório como infraestrutura e utilizando DuckDB + LanceDB para persistência GitOps. O diferencial está na engenharia de contexto inspirada no Claude-Flow, permitindo memória de longo prazo e raciocínio contextualizado sobre papers extensos.

---

## 1. Arquitetura de Alto Nível

### 1.1 Topologia Multi-Agentes

O sistema é composto por cinco agentes especializados que colaboram de forma orquestrada:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ARXIV AGENT ORCHESTRATOR                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│  │    SCOUT     │───▶│   ANALYST    │───▶│    SCRIBE    │                  │
│  │  (Ingestão)  │    │  (Cognitivo) │    │  (Criativo)  │                  │
│  └──────────────┘    └──────────────┘    └──────────────┘                  │
│         │                   │                   │                           │
│         │                   ▼                   │                           │
│         │           ┌──────────────┐            │                           │
│         │           │   CONSENSUS  │            │                           │
│         │           │   ENGINE     │            │                           │
│         │           │  (PBFT/2LLM) │            │                           │
│         │           └──────────────┘            │                           │
│         │                   │                   │                           │
│         ▼                   ▼                   ▼                           │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │                    MEMORY LAYER (Claude-Flow)                    │       │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │       │
│  │  │   DuckDB    │  │   LanceDB   │  │   Context Window Mgr    │  │       │
│  │  │ (Relacional)│  │  (Vetorial) │  │ (Chunking + Hierarchy)  │  │       │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘  │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│         │                                       │                           │
│         ▼                                       ▼                           │
│  ┌──────────────┐                        ┌──────────────┐                  │
│  │   GUARDIAN   │                        │   LIAISON    │                  │
│  │  (Shabbat)   │                        │  (Telegram)  │                  │
│  └──────────────┘                        └──────────────┘                  │
│                                                 │                           │
│                                                 ▼                           │
│                                          ┌──────────────┐                  │
│                                          │   LinkedIn   │                  │
│                                          │   Publisher  │                  │
│                                          └──────────────┘                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Descrição dos Agentes

**Scout (Agente de Ingestão)**
Responsável pela varredura diária do Arxiv, aplicando filtros heurísticos (regex) e semânticos (LLM rápido) para identificar papers relevantes nos tópicos configurados. Detecta sinais de trabalhos disruptivos como AlphaEvolve e Absolute Zero através de padrões específicos.

**Analyst (Agente Cognitivo)**
O cérebro do sistema. Processa papers completos utilizando engenharia de contexto avançada para extrair insights de segunda ordem. Opera em cooperação com o Consensus Engine para validar análises.

**Scribe (Agente Criativo)**
Especialista em copywriting para LinkedIn. Utiliza Few-Shot Learning com posts anteriores do usuário para mimetizar estilo. Gera múltiplas variações para A/B testing implícito.

**Guardian (Agente de Conformidade)**
Módulo determinístico que calcula horários de Shabbat usando astronomia precisa (crate `sunrise`) e bloqueia publicações durante janelas proibidas (sexta ao pôr do sol até sábado ao pôr do sol).

**Liaison (Agente de Interface)**
Gerencia comunicação bidirecional com o humano via Telegram. Envia drafts para aprovação, recebe feedback, e propaga decisões de volta ao sistema.

---

## 2. Fluxo de Dados e Pipeline

### 2.1 Pipeline Principal

```
                                    PIPELINE DIÁRIO
                                    
    ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
    │  Arxiv  │────▶│  Scout  │────▶│ Analyst │────▶│ Scribe  │────▶│Telegram │
    │   API   │     │ Filter  │     │ Process │     │ Generate│     │ Approval│
    └─────────┘     └─────────┘     └─────────┘     └─────────┘     └─────────┘
                          │               │               │               │
                          ▼               ▼               ▼               ▼
                    ┌─────────────────────────────────────────────────────────┐
                    │                    DuckDB / LanceDB                     │
                    │   - papers_processed    - paper_embeddings             │
                    │   - posts_generated     - style_examples               │
                    │   - approval_history    - insights_knowledge           │
                    └─────────────────────────────────────────────────────────┘
                                              │
                                              ▼
                    ┌─────────────────────────────────────────────────────────┐
                    │                 APPROVAL WORKFLOW                       │
                    │                                                         │
                    │  ┌──────────┐   APPROVED   ┌───────────┐               │
                    │  │ Telegram │─────────────▶│  Guardian │──▶ LinkedIn   │
                    │  │ Response │              │   Check   │               │
                    │  └──────────┘              └───────────┘               │
                    │       │                                                 │
                    │       │ REJECTED                                        │
                    │       ▼                                                 │
                    │  ┌──────────┐                                           │
                    │  │  Scribe  │◀─── Feedback Loop                         │
                    │  │ Revise   │                                           │
                    │  └──────────┘                                           │
                    │                                                         │
                    └─────────────────────────────────────────────────────────┘
```

### 2.2 Mecanismo de Consenso entre LLMs

O sistema utiliza um protocolo inspirado em PBFT (Practical Byzantine Fault Tolerance) simplificado para decisões críticas entre os dois LLMs:

```
                         CONSENSUS PROTOCOL
                         
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  ┌─────────────┐         ┌─────────────┐                       │
    │  │    Groq     │         │ OpenRouter  │                       │
    │  │ gpt-oss-120b│         │ grok-4.1    │                       │
    │  └──────┬──────┘         └──────┬──────┘                       │
    │         │                       │                               │
    │         ▼                       ▼                               │
    │  ┌─────────────┐         ┌─────────────┐                       │
    │  │  Proposal   │         │  Proposal   │                       │
    │  │  (Draft A)  │         │  (Draft B)  │                       │
    │  └──────┬──────┘         └──────┬──────┘                       │
    │         │                       │                               │
    │         └───────────┬───────────┘                               │
    │                     ▼                                           │
    │              ┌─────────────┐                                    │
    │              │   COMPARE   │                                    │
    │              │   & MERGE   │                                    │
    │              └──────┬──────┘                                    │
    │                     │                                           │
    │         ┌───────────┼───────────┐                               │
    │         ▼           ▼           ▼                               │
    │   ┌───────────┐ ┌───────────┐ ┌───────────┐                    │
    │   │ Agreement │ │ Partial   │ │ Conflict  │                    │
    │   │ (Use as-is)│ │(Merge)   │ │ (Re-vote) │                    │
    │   └───────────┘ └───────────┘ └───────────┘                    │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

**Regras do Consenso:**
1. Ambos LLMs recebem o mesmo contexto e prompt
2. Cada um gera sua proposta independente
3. As propostas são comparadas por similaridade semântica
4. Se concordância > 80%: aceita proposta com maior score interno
5. Se concordância 50-80%: merge inteligente das melhores partes
6. Se concordância < 50%: terceira rodada com contexto adicional

---

## 3. Engenharia de Contexto (Claude-Flow)

### 3.1 Arquitetura de Memória Híbrida

O sistema implementa o paradigma Claude-Flow adaptado para Rust, combinando diferentes tipos de memória:

```
                    HYBRID MEMORY ARCHITECTURE
                    
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │                  EPISODIC MEMORY                        │   │
    │  │              (DuckDB: session_logs)                     │   │
    │  │                                                         │   │
    │  │  - Current session state                                │   │
    │  │  - Papers processed today                               │   │
    │  │  - API errors and retries                               │   │
    │  │  - Decisions made this run                              │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │                  SEMANTIC MEMORY                        │   │
    │  │             (LanceDB: paper_embeddings)                 │   │
    │  │                                                         │   │
    │  │  - Vector embeddings of all processed papers            │   │
    │  │  - Similarity search for deduplication                  │   │
    │  │  - Related work retrieval                               │   │
    │  │  - Style examples embeddings                            │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │                 REASONING BANK                          │   │
    │  │           (DuckDB: insights_knowledge)                  │   │
    │  │                                                         │   │
    │  │  - Paper → Insight mappings                             │   │
    │  │  - User feedback (approved/rejected)                    │   │
    │  │  - Quality scores over time                             │   │
    │  │  - Learned preferences                                  │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │              CONTEXT WINDOW MANAGER                     │   │
    │  │                                                         │   │
    │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │   │
    │  │  │Chunking │  │Hierarchy│  │Priority │                 │   │
    │  │  │ Engine  │  │ Builder │  │ Queue   │                 │   │
    │  │  └─────────┘  └─────────┘  └─────────┘                 │   │
    │  │                                                         │   │
    │  │  - Papers divididos em chunks de ~4K tokens             │   │
    │  │  - Hierarquia: Summary → Section → Detail               │   │
    │  │  - Retrieval dinâmico baseado na query                  │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

### 3.2 Estratégia para Papers Extensos (100+ páginas)

Para papers muito grandes, o sistema implementa processamento hierárquico:

```
                    HIERARCHICAL PAPER PROCESSING
                    
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  PAPER (200 páginas)                                           │
    │         │                                                       │
    │         ▼                                                       │
    │  ┌─────────────┐                                               │
    │  │   LEVEL 0   │  Full PDF extraction + structure detection    │
    │  │  (Raw Text) │  Identificar seções, figuras, tabelas         │
    │  └──────┬──────┘                                               │
    │         │                                                       │
    │         ▼                                                       │
    │  ┌─────────────┐                                               │
    │  │   LEVEL 1   │  Resumo executivo de cada seção (~500 tokens) │
    │  │  (Summaries)│  Gerado por Groq (rápido)                     │
    │  └──────┬──────┘                                               │
    │         │                                                       │
    │         ▼                                                       │
    │  ┌─────────────┐                                               │
    │  │   LEVEL 2   │  Análise profunda das seções-chave            │
    │  │  (Analysis) │  Gerado por OpenRouter (raciocínio)           │
    │  └──────┬──────┘                                               │
    │         │                                                       │
    │         ▼                                                       │
    │  ┌─────────────┐                                               │
    │  │   LEVEL 3   │  Síntese final + insights de 2ª ordem         │
    │  │ (Synthesis) │  Consenso entre LLMs                          │
    │  └──────┬──────┘                                               │
    │         │                                                       │
    │         ▼                                                       │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │                POST SERIES PLANNING                      │   │
    │  │                                                          │   │
    │  │   Paper muito grande? → Dividir em série de posts       │   │
    │  │                                                          │   │
    │  │   Post 1 (Pt 1): Problema e Motivação                   │   │
    │  │   Post 2 (Pt 2): Solução Técnica Principal              │   │
    │  │   Post 3 (Pt 3): Resultados e Implicações               │   │
    │  │   Post 4 (Pt N): Conexões e Futuro                      │   │
    │  │                                                          │   │
    │  │   Cada post é armazenado com series_id para tracking    │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

---

## 4. Rate Limiting e Gerenciamento de APIs

### 4.1 Limites e Estratégia de Uso

```
                    RATE LIMIT MANAGEMENT
                    
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  GROQ (openai/gpt-oss-120b)                                    │
    │  ├── RPM: 30 requests/minute                                   │
    │  ├── RPD: 1,000 requests/day                                   │
    │  ├── TPM: 8,000 tokens/minute                                  │
    │  └── TPD: 200,000 tokens/day                                   │
    │                                                                 │
    │  USO PLANEJADO:                                                │
    │  ├── Filtragem inicial: ~50 papers × 500 tokens = 25K tokens   │
    │  ├── Análise rápida: ~10 papers × 2K tokens = 20K tokens       │
    │  ├── Consenso: ~5 rounds × 3K tokens = 15K tokens              │
    │  └── Buffer diário: ~140K tokens restantes                     │
    │                                                                 │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                 │
    │  OPENROUTER (x-ai/grok-4.1-fast:free)                          │
    │  └── 1,000 requests/day (tag :free)                            │
    │                                                                 │
    │  USO PLANEJADO:                                                │
    │  ├── Análise profunda: ~3 papers × 5 req = 15 requests         │
    │  ├── Geração de post: ~3 variações × 2 req = 6 requests        │
    │  ├── Consenso: ~10 rounds × 2 req = 20 requests                │
    │  └── Buffer diário: ~950 requests restantes                    │
    │                                                                 │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                 │
    │  RATE LIMITER IMPLEMENTATION                                   │
    │                                                                 │
    │  struct RateLimiter {                                          │
    │      provider: Provider,                                        │
    │      window_requests: AtomicU32,                               │
    │      window_tokens: AtomicU32,                                 │
    │      daily_requests: AtomicU32,                                │
    │      daily_tokens: AtomicU32,                                  │
    │      last_reset: Instant,                                      │
    │  }                                                              │
    │                                                                 │
    │  - Persiste contadores em DuckDB entre execuções               │
    │  - Fallback automático: Groq → OpenRouter → Pause              │
    │  - Exponential backoff em 429s                                 │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

### 4.2 Dashboard de Monitoramento

O sistema inclui um dashboard HTML estático gerado a cada execução:

```
                    MONITORING DASHBOARD
                    
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  GERADO POR: maud (HTML type-safe em Rust)                     │
    │  LOCALIZAÇÃO: ./dashboard/index.html                            │
    │  ATUALIZAÇÃO: A cada execução do GitHub Actions                │
    │                                                                 │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │  📊 RATE LIMIT STATUS                                   │   │
    │  │                                                         │   │
    │  │  GROQ                        OPENROUTER                 │   │
    │  │  ████████░░ 80%              ██░░░░░░░░ 20%            │   │
    │  │  800/1000 req                200/1000 req               │   │
    │  │  160K/200K tokens            N/A tokens                 │   │
    │  │                                                         │   │
    │  │  Reset: 4h 23m               Reset: 4h 23m              │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │  📈 ÚLTIMAS 7 EXECUÇÕES                                 │   │
    │  │                                                         │   │
    │  │  Data       Papers  Posts  Aprovados  Erros            │   │
    │  │  Nov 24     12      2      2          0                 │   │
    │  │  Nov 23     15      1      1          0                 │   │
    │  │  Nov 22     8       1      0          1 (rejected)      │   │
    │  │  ...                                                    │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │  🛡️ SEGURANÇA                                           │   │
    │  │                                                         │   │
    │  │  Prompt Injection Attempts: 0                           │   │
    │  │  Jailbreak Attempts: 0                                  │   │
    │  │  Last Security Scan: Nov 24, 2025 06:00 UTC             │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

---

## 5. Segurança: Proteção contra Jailbreak e Prompt Injection

### 5.1 Arquitetura de Defesa em Camadas

```
                    SECURITY ARCHITECTURE
                    
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  LAYER 1: INPUT SANITIZATION                                   │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │  - Strip caracteres de controle                         │   │
    │  │  - Normalizar Unicode (NFKC)                            │   │
    │  │  - Detectar padrões de injection conhecidos             │   │
    │  │  - Limitar tamanho de input                             │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    │  LAYER 2: PROMPT HARDENING                                     │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │  - System prompts imutáveis (hardcoded)                 │   │
    │  │  - Delimitadores claros para user content               │   │
    │  │  - Instruções de role enforcement                       │   │
    │  │  - Output format constraints                            │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    │  LAYER 3: OUTPUT VALIDATION                                    │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │  - Schema validation (JSON structured output)           │   │
    │  │  - Content policy checks                                │   │
    │  │  - Length bounds verification                           │   │
    │  │  - Blocklist de termos proibidos                        │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    │  LAYER 4: BEHAVIORAL MONITORING                                │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │  - Log all LLM interactions                             │   │
    │  │  - Anomaly detection em patterns de uso                 │   │
    │  │  - Rate limiting por tipo de operação                   │   │
    │  │  - Human review para outputs suspeitos                  │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

### 5.2 Implementação de Sanitização

```rust
// Exemplo conceitual de sanitização em Rust

pub struct InputSanitizer {
    injection_patterns: Vec<Regex>,
    max_length: usize,
}

impl InputSanitizer {
    pub fn sanitize(&self, input: &str) -> Result<String, SecurityError> {
        // 1. Normalize Unicode
        let normalized = input.nfkc().collect::<String>();
        
        // 2. Remove control characters
        let cleaned = normalized.chars()
            .filter(|c| !c.is_control() || *c == '\n')
            .collect::<String>();
        
        // 3. Check length
        if cleaned.len() > self.max_length {
            return Err(SecurityError::InputTooLong);
        }
        
        // 4. Check for injection patterns
        for pattern in &self.injection_patterns {
            if pattern.is_match(&cleaned) {
                log::warn!("Injection pattern detected: {}", pattern);
                return Err(SecurityError::InjectionDetected);
            }
        }
        
        Ok(cleaned)
    }
}
```

---

## 6. Estrutura do Projeto

### 6.1 Árvore de Diretórios

```
arxiv-linkedin-agent/
├── .github/
│   └── workflows/
│       ├── daily-scan.yml          # Workflow principal (6x/dia)
│       ├── approval-check.yml      # Verifica aprovações Telegram
│       ├── publish.yml             # Publica posts aprovados
│       └── backup.yml              # Backup semanal do DuckDB
│
├── src/
│   ├── main.rs                     # Entry point
│   │
│   ├── agents/
│   │   ├── mod.rs
│   │   ├── scout.rs                # Agente de ingestão Arxiv
│   │   ├── analyst.rs              # Agente cognitivo
│   │   ├── scribe.rs               # Agente de escrita
│   │   ├── guardian.rs             # Agente de conformidade Shabbat
│   │   └── liaison.rs              # Agente de interface Telegram
│   │
│   ├── consensus/
│   │   ├── mod.rs
│   │   ├── pbft.rs                 # Protocolo de consenso
│   │   └── merger.rs               # Merge de propostas
│   │
│   ├── memory/
│   │   ├── mod.rs
│   │   ├── duckdb.rs               # Interface DuckDB
│   │   ├── lancedb.rs              # Interface LanceDB
│   │   ├── context_manager.rs      # Gerenciador de janela de contexto
│   │   └── embeddings.rs           # Geração de embeddings
│   │
│   ├── llm/
│   │   ├── mod.rs
│   │   ├── provider.rs             # Trait para providers
│   │   ├── groq.rs                 # Cliente Groq
│   │   ├── openrouter.rs           # Cliente OpenRouter
│   │   └── rate_limiter.rs         # Rate limiting
│   │
│   ├── integrations/
│   │   ├── mod.rs
│   │   ├── arxiv.rs                # Cliente API Arxiv
│   │   ├── linkedin.rs             # Cliente API LinkedIn
│   │   └── telegram.rs             # Bot Telegram
│   │
│   ├── security/
│   │   ├── mod.rs
│   │   ├── sanitizer.rs            # Sanitização de input
│   │   ├── prompt_guard.rs         # Proteção de prompts
│   │   └── output_validator.rs     # Validação de output
│   │
│   ├── observability/
│   │   ├── mod.rs
│   │   ├── tracing.rs              # Distributed tracing
│   │   ├── metrics.rs              # Métricas de execução
│   │   └── dashboard.rs            # Gerador de dashboard HTML
│   │
│   ├── config/
│   │   ├── mod.rs
│   │   └── settings.rs             # Configurações tipadas
│   │
│   └── utils/
│       ├── mod.rs
│       ├── pdf_parser.rs           # Parser de PDFs
│       └── text_chunker.rs         # Chunking de texto
│
├── data/
│   ├── memory.duckdb               # Banco principal (gitignore)
│   ├── embeddings/                 # Índices LanceDB (gitignore)
│   └── style_examples/             # Posts de exemplo do usuário
│
├── dashboard/
│   └── index.html                  # Dashboard estático gerado
│
├── config/
│   ├── topics.toml                 # Tópicos de interesse configuráveis
│   ├── prompts.toml                # Prompts do sistema
│   └── linkedin_profiles.toml      # Perfis LinkedIn configurados
│
├── Cargo.toml
├── Cargo.lock
├── README.md
├── LICENSE (MIT)
└── .env.example
```

### 6.2 Dependências (Cargo.toml)

```toml
[package]
name = "arxiv-linkedin-agent"
version = "0.1.0"
edition = "2024"
rust-version = "1.82"
license = "MIT"
description = "Multi-agent system for AI paper curation and LinkedIn publishing"
repository = "https://github.com/SamoraDC/arxiv-linkedin-agent"
authors = ["Davi Samora <samora.davi@gmail.com>"]

[dependencies]
# Async Runtime
tokio = { version = "1.40", features = ["full"] }

# LLM & AI
rig-core = "0.6"              # Orquestração de agentes
async-openai = "0.25"         # Cliente OpenAI-compatible (Groq/OpenRouter)
fastembed = "4.0"             # Embeddings locais

# Databases
duckdb = { version = "1.4", features = ["bundled", "json", "parquet"] }
lancedb = "0.15"              # Vector database
arrow = "53"                  # Apache Arrow

# HTTP & APIs
reqwest = { version = "0.12", features = ["json", "cookies", "gzip"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
serde_xml_rs = "0.6"          # Parser Arxiv XML

# Telegram
teloxide = { version = "0.14", features = ["macros", "auto-send"] }

# Date/Time & Astronomy
chrono = { version = "0.4", features = ["serde"] }
sunrise = "1.2"               # Cálculo de Shabbat

# HTML Generation
maud = "0.26"                 # HTML type-safe

# Observability
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
opentelemetry = "0.27"
metrics = "0.24"

# Security
regex = "1.10"
unicode-normalization = "0.1"

# Utils
anyhow = "1.0"
thiserror = "2.0"
config = "0.14"
toml = "0.8"
uuid = { version = "1.10", features = ["v4", "serde"] }
base64 = "0.22"

# PDF Processing
pdf-extract = "0.7"
lopdf = "0.34"

[dev-dependencies]
tokio-test = "0.4"
mockall = "0.13"
proptest = "1.5"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
strip = true
```

---

## 7. GitHub Actions Workflows

### 7.1 Workflow Principal (daily-scan.yml)

```yaml
name: Daily ArXiv Scan

on:
  schedule:
    # Executa 4x por dia (exceto sexta 18h - sábado 18h, verificado no código)
    - cron: '0 6,12,18,0 * * *'
  workflow_dispatch:
    inputs:
      force_run:
        description: 'Force run even during Shabbat'
        required: false
        default: 'false'

env:
  CARGO_TERM_COLOR: always
  RUST_BACKTRACE: 1

jobs:
  scan-and-process:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Setup Rust
        uses: dtolnay/rust-action@stable
        with:
          toolchain: stable
          components: clippy
          
      - name: Cache Cargo
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/bin/
            ~/.cargo/registry/index/
            ~/.cargo/registry/cache/
            ~/.cargo/git/db/
            target/
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
          
      - name: Download Data Artifacts
        uses: actions/download-artifact@v4
        with:
          name: agent-data
          path: data/
        continue-on-error: true  # First run won't have artifacts
        
      - name: Build Release Binary
        run: cargo build --release
        
      - name: Run Agent
        env:
          GROQ_API_KEY: ${{ secrets.GROQ_API_KEY }}
          OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
          TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
          TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
          LINKEDIN_ACCESS_TOKEN: ${{ secrets.LINKEDIN_ACCESS_TOKEN }}
          LINKEDIN_PERSON_URN: ${{ secrets.LINKEDIN_PERSON_URN }}
          USER_LATITUDE: ${{ secrets.USER_LATITUDE }}
          USER_LONGITUDE: ${{ secrets.USER_LONGITUDE }}
        run: ./target/release/arxiv-linkedin-agent scan
        
      - name: Upload Data Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: agent-data
          path: |
            data/memory.duckdb
            data/embeddings/
          retention-days: 90
          
      - name: Upload Dashboard
        uses: actions/upload-pages-artifact@v3
        with:
          path: dashboard/
          
      - name: Commit Dashboard Updates
        run: |
          git config user.name 'ArxivAgent Bot'
          git config user.email 'bot@arxivagent.local'
          git add dashboard/
          git diff --staged --quiet || git commit -m "📊 Update dashboard $(date +%Y-%m-%d)"
          git push
```

### 7.2 Workflow de Aprovação (approval-check.yml)

```yaml
name: Check Telegram Approvals

on:
  schedule:
    - cron: '*/15 * * * *'  # A cada 15 minutos
  workflow_dispatch:

jobs:
  check-approvals:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Download Data
        uses: actions/download-artifact@v4
        with:
          name: agent-data
          path: data/
          
      - name: Setup Rust
        uses: dtolnay/rust-action@stable
        
      - name: Build
        run: cargo build --release
        
      - name: Check Approvals
        env:
          TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
          TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
        run: ./target/release/arxiv-linkedin-agent check-approvals
        
      - name: Upload Updated Data
        uses: actions/upload-artifact@v4
        with:
          name: agent-data
          path: data/
```

### 7.3 Workflow de Publicação (publish.yml)

```yaml
name: Publish Approved Posts

on:
  workflow_run:
    workflows: ["Check Telegram Approvals"]
    types: [completed]

jobs:
  publish:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Download Data
        uses: actions/download-artifact@v4
        with:
          name: agent-data
          path: data/
          
      - name: Setup Rust
        uses: dtolnay/rust-action@stable
        
      - name: Build
        run: cargo build --release
        
      - name: Publish to LinkedIn
        env:
          LINKEDIN_ACCESS_TOKEN: ${{ secrets.LINKEDIN_ACCESS_TOKEN }}
          LINKEDIN_PERSON_URN: ${{ secrets.LINKEDIN_PERSON_URN }}
          USER_LATITUDE: ${{ secrets.USER_LATITUDE }}
          USER_LONGITUDE: ${{ secrets.USER_LONGITUDE }}
        run: ./target/release/arxiv-linkedin-agent publish
        
      - name: Notify Success
        env:
          TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
          TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
        run: ./target/release/arxiv-linkedin-agent notify-success
```

---

## 8. Schemas do Banco de Dados

### 8.1 DuckDB Schema

```sql
-- Tabela principal de papers processados
CREATE TABLE papers (
    id VARCHAR PRIMARY KEY,           -- arxiv_id (e.g., "2505.12345")
    title VARCHAR NOT NULL,
    abstract TEXT NOT NULL,
    authors JSON NOT NULL,            -- ["Author 1", "Author 2"]
    categories JSON NOT NULL,         -- ["cs.AI", "cs.LG"]
    published_date DATE NOT NULL,
    pdf_url VARCHAR,
    source_url VARCHAR NOT NULL,
    
    -- Processamento
    processed_at TIMESTAMP NOT NULL,
    relevance_score FLOAT,            -- 0-10
    priority VARCHAR,                 -- 'normal', 'high', 'critical'
    
    -- Estado
    status VARCHAR NOT NULL,          -- 'filtered', 'analyzed', 'posted', 'skipped'
    
    -- Metadados
    full_text_extracted BOOLEAN DEFAULT FALSE,
    page_count INTEGER,
    
    UNIQUE(id)
);

-- Posts gerados
CREATE TABLE posts (
    id UUID PRIMARY KEY,
    paper_id VARCHAR REFERENCES papers(id),
    
    -- Conteúdo
    content TEXT NOT NULL,
    variation_id INTEGER DEFAULT 1,   -- Para A/B testing
    series_id UUID,                   -- Para posts em série (Pt 1, Pt 2...)
    series_part INTEGER,
    
    -- Geração
    generated_at TIMESTAMP NOT NULL,
    generator_model VARCHAR,          -- 'groq/gpt-oss-120b', 'openrouter/grok-4.1'
    consensus_score FLOAT,            -- Concordância entre LLMs
    
    -- Aprovação
    status VARCHAR NOT NULL,          -- 'pending', 'approved', 'rejected', 'published'
    sent_for_approval_at TIMESTAMP,
    approved_at TIMESTAMP,
    rejection_reason TEXT,
    
    -- Publicação
    published_at TIMESTAMP,
    linkedin_post_urn VARCHAR,
    
    -- Analytics (preenchido após publicação)
    likes_count INTEGER DEFAULT 0,
    comments_count INTEGER DEFAULT 0,
    shares_count INTEGER DEFAULT 0,
    views_count INTEGER DEFAULT 0,
    engagement_rate FLOAT
);

-- Feedback do usuário (Reasoning Bank)
CREATE TABLE feedback (
    id UUID PRIMARY KEY,
    post_id UUID REFERENCES posts(id),
    feedback_type VARCHAR NOT NULL,   -- 'approve', 'reject', 'edit'
    feedback_text TEXT,
    received_at TIMESTAMP NOT NULL
);

-- Sessões de execução (Episodic Memory)
CREATE TABLE sessions (
    id UUID PRIMARY KEY,
    started_at TIMESTAMP NOT NULL,
    ended_at TIMESTAMP,
    
    -- Métricas
    papers_scanned INTEGER DEFAULT 0,
    papers_filtered INTEGER DEFAULT 0,
    papers_analyzed INTEGER DEFAULT 0,
    posts_generated INTEGER DEFAULT 0,
    
    -- Rate limits
    groq_requests_used INTEGER DEFAULT 0,
    groq_tokens_used INTEGER DEFAULT 0,
    openrouter_requests_used INTEGER DEFAULT 0,
    
    -- Erros
    errors JSON,
    
    -- Status
    status VARCHAR NOT NULL           -- 'running', 'completed', 'failed'
);

-- Exemplos de estilo do usuário
CREATE TABLE style_examples (
    id UUID PRIMARY KEY,
    content TEXT NOT NULL,
    source VARCHAR,                   -- 'linkedin', 'manual'
    added_at TIMESTAMP NOT NULL,
    engagement_score FLOAT,           -- Para priorizar exemplos de sucesso
    
    -- Embedding é armazenado no LanceDB
    embedding_id VARCHAR
);

-- Rate limits persistidos
CREATE TABLE rate_limits (
    provider VARCHAR PRIMARY KEY,     -- 'groq', 'openrouter'
    requests_used INTEGER DEFAULT 0,
    tokens_used INTEGER DEFAULT 0,
    last_reset TIMESTAMP NOT NULL,
    
    -- Histórico
    daily_history JSON                -- Últimos 7 dias
);

-- Configurações dinâmicas
CREATE TABLE config (
    key VARCHAR PRIMARY KEY,
    value JSON NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

### 8.2 LanceDB Schema (Vector Store)

```python
# Conceitual - implementado em Rust via lancedb crate

# Tabela de embeddings de papers
paper_embeddings = {
    "paper_id": str,           # FK para DuckDB
    "vector": Vector(384),     # fastembed output dimension
    "text_chunk": str,         # Texto original do chunk
    "chunk_type": str,         # 'abstract', 'introduction', 'methodology', etc.
    "chunk_index": int
}

# Tabela de embeddings de estilo
style_embeddings = {
    "example_id": str,         # FK para DuckDB
    "vector": Vector(384),
    "text": str
}

# Tabela de embeddings de insights
insight_embeddings = {
    "insight_id": str,
    "paper_id": str,
    "vector": Vector(384),
    "insight_text": str,
    "quality_score": float     # Baseado em feedback
}
```

---

## 9. Interface de Configuração

### 9.1 Arquivo de Tópicos (config/topics.toml)

```toml
[topics]
# Tópicos primários (sempre buscados)
primary = [
    "reinforcement learning",
    "multi-agent systems",
    "RAG retrieval augmented generation",
    "R3 retrieval reasoning reinforcement",
    "AlphaEvolve",
    "Absolute Zero",
    "context engineering",
    "prompt engineering",
    "agentic AI",
    "LLM agents"
]

# Tópicos secundários (buscados se houver quota)
secondary = [
    "machine learning",
    "deep learning",
    "neural networks",
    "transformer architectures",
    "self-play",
    "RLHF",
    "DPO",
    "chain of thought",
    "reasoning",
    "code generation"
]

# Tópicos de exclusão
exclude = [
    "biology",
    "chemistry",
    "physics",
    "medical"
]

[arxiv]
# Categorias do Arxiv a monitorar
categories = ["cs.AI", "cs.LG", "cs.CL", "cs.MA", "cs.NE"]

# Máximo de papers por dia
max_papers_per_day = 50

# Máximo de papers a analisar profundamente
max_deep_analysis = 5

[detection]
# Padrões para detectar papers disruptivos
disruptive_patterns = [
    { name = "AlphaEvolve", keywords = ["evolutionary", "code generation", "automated discovery"] },
    { name = "Absolute Zero", keywords = ["zero data", "self-play reasoning", "no training data"] },
    { name = "Novel Architecture", keywords = ["new architecture", "breakthrough", "state-of-the-art"] }
]
```

### 9.2 Arquivo de Perfis LinkedIn (config/linkedin_profiles.toml)

```toml
# Perfis configuráveis para diferentes usuários

[[profiles]]
name = "default"
person_urn = "${LINKEDIN_PERSON_URN}"  # Variável de ambiente
access_token = "${LINKEDIN_ACCESS_TOKEN}"

# Preferências de estilo
style = """
Profissional mas acessível
Parágrafos curtos
Uso moderado de emojis
Foco em insights práticos
Conexões com indústria
"""

# Horários preferenciais de postagem
preferred_hours = [9, 12, 18]

# Hashtags padrão
default_hashtags = ["#AI", "#MachineLearning", "#DataScience", "#Tech"]

[[profiles]]
name = "secondary"
# Pode ter outro perfil configurado
```

### 9.3 Interface de Setup Inicial

O sistema inclui um comando de setup interativo:

```bash
$ ./arxiv-linkedin-agent setup

🚀 ArxivAgent Setup Wizard

📍 Step 1/5: API Keys
   Enter your Groq API Key: ********
   ✅ Groq API verified (model: openai/gpt-oss-120b available)
   
   Enter your OpenRouter API Key: ********
   ✅ OpenRouter API verified (model: x-ai/grok-4.1-fast:free available)

📱 Step 2/5: Telegram Bot
   Enter your Telegram Bot Token: ********
   ✅ Bot verified (@YourBotName)
   
   Enter your Telegram Chat ID: ********
   ✅ Chat access verified

💼 Step 3/5: LinkedIn
   Follow these steps to get your LinkedIn access:
   1. Go to https://www.linkedin.com/developers/apps
   2. Create a new app
   3. Request 'Share on LinkedIn' permission
   4. Generate access token
   
   Enter your LinkedIn Access Token: ********
   ✅ LinkedIn API verified
   
   Enter your LinkedIn Person URN (from profile): ********
   ✅ Profile access verified

📍 Step 4/5: Location (for Shabbat calculation)
   Enter your latitude (e.g., -22.9068): -22.9068
   Enter your longitude (e.g., -43.1729): -43.1729
   ✅ Location: Campinas, SP, Brazil

📝 Step 5/5: Style Examples
   Would you like to import your recent LinkedIn posts? (y/n): y
   ✅ Imported 15 posts as style examples

✨ Setup complete! Your secrets have been saved to .env
   
   Next steps:
   1. Add secrets to GitHub repository settings
   2. Run: ./arxiv-linkedin-agent test
   3. Enable GitHub Actions workflows
```

---

## 10. Observabilidade e Rastreabilidade

### 10.1 Arquitetura de Tracing

```
                    OBSERVABILITY STACK
                    
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  APPLICATION LAYER                                              │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │                     tracing crate                        │   │
    │  │                                                          │   │
    │  │  #[instrument(skip(self), fields(paper_id = %paper.id))] │   │
    │  │  async fn analyze_paper(&self, paper: &Paper) { ... }    │   │
    │  │                                                          │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │         │                                                       │
    │         ▼                                                       │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │              tracing-subscriber                          │   │
    │  │                                                          │   │
    │  │  - JSON formatted logs                                   │   │
    │  │  - Structured fields (paper_id, agent, model, etc.)     │   │
    │  │  - Span timing                                           │   │
    │  │  - Error context                                         │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │         │                                                       │
    │         ▼                                                       │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │                    OUTPUTS                               │   │
    │  │                                                          │   │
    │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
    │  │  │  STDOUT  │  │  DuckDB  │  │Dashboard │              │   │
    │  │  │  (JSON)  │  │  (logs)  │  │  (HTML)  │              │   │
    │  │  └──────────┘  └──────────┘  └──────────┘              │   │
    │  │       │              │              │                    │   │
    │  │       ▼              ▼              ▼                    │   │
    │  │  GH Actions     Queries SQL    GitHub Pages             │   │
    │  │    Logs                                                  │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

### 10.2 Métricas Coletadas

```rust
// Métricas estruturadas registradas a cada execução

pub struct ExecutionMetrics {
    // Timing
    pub total_duration_ms: u64,
    pub arxiv_fetch_ms: u64,
    pub llm_processing_ms: u64,
    pub embedding_generation_ms: u64,
    
    // Volumes
    pub papers_fetched: u32,
    pub papers_filtered_heuristic: u32,
    pub papers_filtered_semantic: u32,
    pub papers_analyzed: u32,
    pub posts_generated: u32,
    
    // API Usage
    pub groq_requests: u32,
    pub groq_tokens_input: u32,
    pub groq_tokens_output: u32,
    pub openrouter_requests: u32,
    
    // Quality
    pub average_relevance_score: f32,
    pub consensus_agreement_rate: f32,
    
    // Errors
    pub api_errors: u32,
    pub retry_count: u32,
    pub rate_limit_hits: u32,
}
```

---

## 11. Backup e Disaster Recovery

### 11.1 Estratégia de Backup

```
                    BACKUP ARCHITECTURE
                    
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  PRIMARY STORAGE                                                │
    │  ├── GitHub Artifacts (90 days retention)                      │
    │  │   └── data/memory.duckdb                                    │
    │  │   └── data/embeddings/                                      │
    │  │                                                              │
    │  SECONDARY BACKUP (Weekly)                                      │
    │  ├── Google Drive (via rclone)                                 │
    │  │   └── arxiv-agent-backup-YYYY-MM-DD.tar.gz                 │
    │  │                                                              │
    │  CLEANUP POLICY                                                 │
    │  ├── Remove embeddings de papers > 6 meses                     │
    │  ├── Manter títulos de todos os papers (deduplicação)         │
    │  ├── Comprimir logs > 30 dias                                  │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

### 11.2 Workflow de Backup (backup.yml)

```yaml
name: Weekly Backup

on:
  schedule:
    - cron: '0 3 * * 0'  # Domingos às 3h
  workflow_dispatch:

jobs:
  backup:
    runs-on: ubuntu-latest
    
    steps:
      - name: Download Data
        uses: actions/download-artifact@v4
        with:
          name: agent-data
          path: data/
          
      - name: Setup rclone
        uses: animosity22/gclone-action@v1
        
      - name: Configure rclone
        run: |
          mkdir -p ~/.config/rclone
          echo "${{ secrets.RCLONE_CONFIG }}" > ~/.config/rclone/rclone.conf
          
      - name: Create Backup Archive
        run: |
          DATE=$(date +%Y-%m-%d)
          tar -czvf backup-$DATE.tar.gz data/
          
      - name: Upload to Google Drive
        run: |
          rclone copy backup-*.tar.gz gdrive:arxiv-agent-backups/
          
      - name: Cleanup Old Backups
        run: |
          # Manter apenas últimos 4 backups
          rclone delete gdrive:arxiv-agent-backups/ --min-age 30d
```

---

## 12. A/B Testing e Analytics

### 12.1 Sistema de Variações

```
                    A/B TESTING FLOW
                    
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  GERAÇÃO DE VARIAÇÕES                                          │
    │                                                                 │
    │  Paper Selecionado                                              │
    │         │                                                       │
    │         ▼                                                       │
    │  ┌─────────────┐                                               │
    │  │   Scribe    │                                               │
    │  │   Agent     │                                               │
    │  └──────┬──────┘                                               │
    │         │                                                       │
    │         ├──────────────┬──────────────┐                        │
    │         ▼              ▼              ▼                        │
    │  ┌───────────┐  ┌───────────┐  ┌───────────┐                  │
    │  │Variation A│  │Variation B│  │Variation C│                  │
    │  │(Técnico)  │  │(Storytell)│  │(Provocativo)                │
    │  └───────────┘  └───────────┘  └───────────┘                  │
    │         │              │              │                        │
    │         └──────────────┼──────────────┘                        │
    │                        ▼                                        │
    │                 ┌─────────────┐                                 │
    │                 │   Ranking   │                                 │
    │                 │  Internal   │                                 │
    │                 └──────┬──────┘                                 │
    │                        │                                        │
    │                        ▼                                        │
    │                 ┌─────────────┐                                 │
    │                 │   Telegram  │  "Qual variação prefere?"      │
    │                 │   Choice    │  [A] [B] [C]                   │
    │                 └─────────────┘                                 │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

### 12.2 Coleta de Analytics (Pós-Publicação)

```
                    ENGAGEMENT TRACKING
                    
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  POST PUBLICADO                                                 │
    │         │                                                       │
    │         ▼                                                       │
    │  ┌─────────────┐                                               │
    │  │  LinkedIn   │  URN: urn:li:share:1234567890                 │
    │  │  Post URN   │                                               │
    │  └──────┬──────┘                                               │
    │         │                                                       │
    │         │  (24h depois)                                        │
    │         ▼                                                       │
    │  ┌─────────────┐                                               │
    │  │  Analytics  │  GET /socialActions/{urn}/metadata            │
    │  │  API Call   │                                               │
    │  └──────┬──────┘                                               │
    │         │                                                       │
    │         ▼                                                       │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │  MÉTRICAS COLETADAS                                     │   │
    │  │                                                         │   │
    │  │  - likes_count                                          │   │
    │  │  - comments_count                                       │   │
    │  │  - shares_count                                         │   │
    │  │  - impressions (se disponível)                          │   │
    │  │  - engagement_rate = (likes+comments+shares)/impressions│   │
    │  └─────────────────────────────────────────────────────────┘   │
    │         │                                                       │
    │         ▼                                                       │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │  REINFORCEMENT LEARNING SIGNAL                          │   │
    │  │                                                         │   │
    │  │  - Update style_examples with engagement scores         │   │
    │  │  - Priorizar variações com alto engagement              │   │
    │  │  - Ajustar prompts do Scribe baseado em feedback        │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

---

## 13. Roadmap de Implementação

### Fase 1: Core Infrastructure (Semanas 1-2)

```
□ Setup inicial do projeto Rust
□ Implementar cliente Arxiv API (parsing XML)
□ Implementar cliente DuckDB (schema inicial)
□ Implementar cliente Groq (async-openai)
□ Implementar cliente OpenRouter
□ Rate limiter básico
□ Testes unitários de cada componente
```

### Fase 2: Agents & Memory (Semanas 3-4)

```
□ Implementar Scout Agent (filtragem heurística + semântica)
□ Implementar sistema de embeddings (fastembed)
□ Integrar LanceDB para busca vetorial
□ Implementar Context Window Manager
□ Implementar Analyst Agent (análise de papers)
□ Implementar lógica de chunking hierárquico
□ Testes de integração
```

### Fase 3: Consensus & Creation (Semanas 5-6)

```
□ Implementar Consensus Engine (PBFT simplificado)
□ Implementar Scribe Agent (geração de posts)
□ Implementar sistema de variações A/B
□ Implementar Few-Shot Learning com exemplos
□ Guardian Agent (cálculo de Shabbat)
□ Testes end-to-end do pipeline
```

### Fase 4: Human Interface (Semanas 7-8)

```
□ Bot Telegram (teloxide)
□ Workflow de aprovação assíncrono
□ Inline keyboards para escolha de variações
□ Feedback loop implementation
□ Cliente LinkedIn API
□ Publicação automática
```

### Fase 5: Observability & Security (Semana 9)

```
□ Tracing distribuído
□ Dashboard HTML (maud)
□ Sanitização de inputs
□ Prompt hardening
□ Output validation
□ Security tests
```

### Fase 6: Deployment & Polish (Semana 10)

```
□ GitHub Actions workflows
□ Backup automation
□ Analytics collection
□ Setup wizard
□ Documentação completa
□ Release v1.0.0
```

---

## 14. Estimativa de Custos

```
                    CUSTO MENSAL ESTIMADO
                    
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  COMPONENTE                              CUSTO                  │
    │  ─────────────────────────────────────────────────────────────  │
    │                                                                 │
    │  GitHub Actions                          $0.00                  │
    │  (2000 min/mês free tier)                                      │
    │                                                                 │
    │  Groq API                                $0.00                  │
    │  (Free tier: 200K tokens/dia)                                  │
    │                                                                 │
    │  OpenRouter                              $0.00                  │
    │  (1000 req/dia com tag :free)                                  │
    │                                                                 │
    │  LinkedIn API                            $0.00                  │
    │  (Free para posts pessoais)                                    │
    │                                                                 │
    │  Telegram Bot                            $0.00                  │
    │  (Gratuito)                                                    │
    │                                                                 │
    │  Storage (GitHub Artifacts)              $0.00                  │
    │  (500MB free)                                                   │
    │                                                                 │
    │  Google Drive Backup                     $0.00                  │
    │  (15GB free)                                                    │
    │                                                                 │
    │  ─────────────────────────────────────────────────────────────  │
    │  TOTAL MENSAL                            $0.00                  │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

---

## 15. Considerações Finais

Este projeto representa uma implementação completa de um sistema multi-agente para curadoria automatizada de conhecimento científico. Os diferenciais principais são:

1. **100% Rust**: Performance, segurança e confiabilidade em ambiente serverless
2. **Custo Zero**: Opera inteiramente dentro de free tiers
3. **Consenso Multi-LLM**: Qualidade garantida por acordo entre modelos
4. **Engenharia de Contexto**: Memória de longo prazo para papers extensos
5. **Human-in-the-Loop**: Controle total sobre publicações via Telegram
6. **Conformidade Religiosa**: Respeito automático ao Shabbat
7. **Observabilidade**: Rastreabilidade completa de ponta a ponta
8. **Segurança**: Proteção contra prompt injection e jailbreak

O sistema está pronto para ser desenvolvido iterativamente, começando pela infraestrutura core e evoluindo até um produto completo e polido.
