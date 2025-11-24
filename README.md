# 🤖 ArxivAgent

[![Rust](https://img.shields.io/badge/Rust-1.82+-orange.svg)](https://www.rust-lang.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![GitHub Actions](https://github.com/SamoraDC/arxiv-linkedin-agent/workflows/Daily%20ArXiv%20Scan/badge.svg)](https://github.com/SamoraDC/arxiv-linkedin-agent/actions)

> **Sistema Multi-Agente para Curadoria Automatizada de Papers de IA e Publicação no LinkedIn**

ArxivAgent é um sistema autônomo construído em Rust que monitora o Arxiv diariamente, identifica papers relevantes sobre IA/ML, gera análises e posts para LinkedIn, e publica automaticamente após aprovação humana via Telegram.

## ✨ Features

- 🔍 **Busca Inteligente**: Filtragem heurística + semântica de papers do Arxiv
- 🧠 **Análise Profunda**: Extração de insights de segunda ordem usando múltiplos LLMs
- 🤝 **Consenso Multi-LLM**: Groq + OpenRouter colaboram via algoritmo PBFT
- 📝 **Geração de Conteúdo**: Posts no estilo do usuário com Few-Shot Learning
- 💾 **Memória Persistente**: Claude-Flow implementado com DuckDB + LanceDB
- 📱 **Human-in-the-Loop**: Aprovação via Telegram com variações A/B
- 🕯️ **Conformidade Shabbat**: Cálculos astronômicos para respeitar janelas religiosas
- 🔒 **Segurança**: Proteção contra prompt injection e jailbreak
- 📊 **Observabilidade**: Tracing distribuído e dashboard de métricas
- 💰 **Custo Zero**: Opera 100% dentro de free tiers

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                      ARXIV AGENT SYSTEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │  Scout   │──▶│ Analyst  │──▶│  Scribe  │──▶│ Liaison  │    │
│  │ (Arxiv)  │   │(Cognitive)│   │(Creative)│   │(Telegram)│    │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘    │
│       │              │              │              │           │
│       └──────────────┴──────────────┴──────────────┘           │
│                            │                                    │
│                            ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              MEMORY LAYER (Claude-Flow)                  │   │
│  │  DuckDB (Relational) + LanceDB (Vector) + Context Mgr   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 🚀 Quick Start

### Pré-requisitos

- Rust 1.82+ (`rustup update stable`)
- Conta no [Groq](https://console.groq.com/) (free tier)
- Conta no [OpenRouter](https://openrouter.ai/) (free tier)
- Bot do Telegram (via [@BotFather](https://t.me/botfather))
- App do LinkedIn Developer

### Instalação

```bash
# Clone o repositório
git clone https://github.com/SamoraDC/arxiv-linkedin-agent.git
cd arxiv-linkedin-agent

# Execute o wizard de setup
cargo run --release -- setup

# Teste a configuração
cargo run --release -- test

# Execute manualmente
cargo run --release -- scan
```

### Configuração via GitHub Secrets

Para deploy automático via GitHub Actions, configure os seguintes secrets:

| Secret | Descrição |
|--------|-----------|
| `GROQ_API_KEY` | Chave da API Groq |
| `OPENROUTER_API_KEY` | Chave da API OpenRouter |
| `TELEGRAM_BOT_TOKEN` | Token do bot Telegram |
| `TELEGRAM_CHAT_ID` | Seu chat ID no Telegram |
| `LINKEDIN_ACCESS_TOKEN` | Token de acesso LinkedIn |
| `LINKEDIN_PERSON_URN` | URN do seu perfil LinkedIn |
| `USER_LATITUDE` | Latitude para cálculo de Shabbat |
| `USER_LONGITUDE` | Longitude para cálculo de Shabbat |

## 📖 Uso

### Comandos Disponíveis

```bash
# Executar scan completo
arxiv-linkedin-agent scan

# Verificar aprovações pendentes no Telegram
arxiv-linkedin-agent check-approvals

# Publicar posts aprovados
arxiv-linkedin-agent publish

# Gerar dashboard de métricas
arxiv-linkedin-agent dashboard

# Setup interativo
arxiv-linkedin-agent setup

# Testar configuração
arxiv-linkedin-agent test
```

### Configuração de Tópicos

Edite `config/topics.toml` para personalizar os tópicos monitorados:

```toml
[topics]
primary = [
    "reinforcement learning",
    "RAG retrieval augmented generation",
    "multi-agent systems",
    # ...
]

[arxiv]
categories = ["cs.AI", "cs.LG", "cs.CL"]
max_papers_per_run = 100
```

## 🔧 Tecnologias

| Componente | Tecnologia | Propósito |
|------------|------------|-----------|
| Linguagem | Rust | Performance e segurança |
| Orquestração | rig-core | Framework de agentes |
| LLM (Velocidade) | Groq (gpt-oss-120b) | Filtragem e consenso |
| LLM (Raciocínio) | OpenRouter (grok-4.1) | Análise profunda |
| Banco Relacional | DuckDB | Persistência estruturada |
| Banco Vetorial | LanceDB | Busca semântica |
| Bot | Teloxide | Interface Telegram |
| Astronomia | sunrise | Cálculo de Shabbat |
| Dashboard | maud | HTML type-safe |

## 📊 Rate Limits

O sistema opera dentro dos seguintes limites gratuitos:

| Provider | Limite | Uso Típico |
|----------|--------|------------|
| Groq | 200K tokens/dia | ~60K tokens/dia |
| OpenRouter | 1000 req/dia | ~50 req/dia |
| GitHub Actions | 2000 min/mês | ~120 min/mês |

## 🛡️ Segurança

- Sanitização de inputs (Unicode normalization, pattern detection)
- Prompts hardcoded (não dinâmicos)
- Validação de outputs (schema, length, blocklist)
- Logs de todas interações LLM
- Secrets via environment variables

## 📈 Roadmap

- [x] Core infrastructure (Arxiv, DuckDB, LLM clients)
- [x] Agent system (Scout, Analyst, Scribe)
- [x] Consensus engine (PBFT)
- [x] Memory layer (Claude-Flow)
- [x] Telegram integration
- [x] LinkedIn publishing
- [ ] A/B testing com analytics
- [ ] RL para otimização de engagement
- [ ] Vision capabilities para análise de figuras

## 🤝 Contribuindo

Contribuições são bem-vindas! Por favor, leia [CONTRIBUTING.md](CONTRIBUTING.md) antes de submeter PRs.

```bash
# Fork o repositório
# Clone seu fork
git clone https://github.com/SEU_USERNAME/arxiv-linkedin-agent.git

# Crie uma branch
git checkout -b feature/minha-feature

# Faça suas mudanças e teste
cargo test
cargo clippy

# Commit e push
git commit -m "feat: minha nova feature"
git push origin feature/minha-feature

# Abra um Pull Request
```

## 📄 Licença

Este projeto está licenciado sob a MIT License - veja [LICENSE](LICENSE) para detalhes.

## 🙏 Agradecimentos

- [Anthropic](https://www.anthropic.com/) - Inspiração do Claude-Flow
- [LangChain](https://langchain.com/) - Conceitos de context engineering
- [Groq](https://groq.com/) - Infraestrutura LPU
- [OpenRouter](https://openrouter.ai/) - Acesso a múltiplos LLMs

---

**Desenvolvido com 🦀 por [Davi Samora](https://www.linkedin.com/in/samoradc/)**

*"Thank you to arXiv for use of its open access interoperability."*
