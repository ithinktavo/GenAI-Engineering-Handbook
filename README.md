# GenAI Engineering Handbook

A practical, hands-on guide to Generative AI engineering — from prompt fundamentals to production-grade agent systems. Built for engineers, architects, and technical leaders who need to understand not just *what* these tools do, but *how* to build with them responsibly in enterprise environments.

---

## Who This Is For

- **Software engineers** transitioning into AI-focused roles
- **Cloud/DevOps engineers** adding GenAI to their toolkit
- **Technical leaders** who need to make architectural decisions about AI systems
- **Consultants and solution architects** working with enterprise and regulated clients
- Anyone who wants to move beyond AI buzzwords and understand the real engineering

## What You'll Learn

This handbook is organized as a progressive learning path — each section builds on the previous one:

### 📘 Part 1 — Foundations
| Guide | Description |
|---|---|
| [From Prompt Engineering to Context Engineering](docs/01-prompt-engineering-to-context-engineering.md) | Role assignment, Chain of Thought, Few-Shot prompting, then the 2026 evolution: memory, state, tools, guardrails, and why prompt engineering is now one component of context engineering |
| [LLMs Explained](docs/03-llms-explained.md) | How language models work, tokens, context windows, temperature, inference — the mental model every engineer needs |

### 🔍 Part 2 — Retrieval & Knowledge
| Guide | Description |
|---|---|
| [RAG Deep Dive](docs/04-rag-deep-dive.md) | The full RAG pipeline: documents, chunking, embeddings, vectors, vector databases, semantic search, and reranking |
| [RAG vs Long Context](docs/05-rag-vs-long-context.md) | The real trade-offs — when to use RAG, when long context wins, and why production teams use both |
| [Common GenAI Misconceptions](docs/06-misconceptions.md) | RAG is not training. More data ≠ better AI. You don't need your own model. The corrections that signal real expertise |

### 🤖 Part 3 — Agents & Orchestration
| Guide | Description |
|---|---|
| [Agentic AI Fundamentals](docs/07-agentic-ai.md) | What agents are, how they differ from chatbots, orchestration patterns (sequential, parallel, supervisor, handoff) |
| [MCP (Model Context Protocol)](docs/08-mcp.md) | The universal connector for AI — what it does, how it works, and the security gaps in enterprise environments |
| [Agent Orchestration Patterns](docs/09-orchestration-patterns.md) | Deep dive into multi-agent architectures, state management, and the Microsoft Agent Framework |

### 🔒 Part 4 — Enterprise & Governance
| Guide | Description |
|---|---|
| [AI Governance in Regulated Environments](docs/10-governance.md) | FedRAMP, HIPAA, PCI-DSS — building AI that auditors can trust |
| [MCP Security for Enterprise](docs/11-mcp-security.md) | The 4 security gaps, 6 enterprise requirements, and Zero Trust patterns for AI tooling |
| [Context Risks & Mitigations](docs/12-context-risks.md) | Context poisoning, distraction, clashing — what goes wrong and how to prevent it |

### 🧪 Part 5 — Hands-On Notebooks
| Notebook | Description |
|---|---|
| [Semantic Search Visualized](notebooks/01-semantic-search-visualized.ipynb) | Interactive visualization of how embeddings and vector similarity work — see semantic search in action |
| [Build a RAG Pipeline](notebooks/02-build-a-rag-pipeline.ipynb) | Build a working RAG system step-by-step: chunk documents, generate embeddings, store in a vector DB, query with an LLM |
| [Chunking Strategies Compared](notebooks/03-chunking-strategies.ipynb) | Experiment with fixed-size, semantic, and overlapping chunking — see how strategy affects retrieval quality |
| [Embeddings Explorer](notebooks/04-embeddings-explorer.ipynb) | Visualize high-dimensional embeddings in 2D/3D space, explore how similar concepts cluster together |
| [RAG vs Long Context Benchmark](notebooks/05-rag-vs-long-context.ipynb) | Run the same queries against RAG and long-context approaches — compare accuracy, latency, and cost |
| [Build a Simple Agent](notebooks/06-build-a-simple-agent.ipynb) | Create an agent that can reason, use tools, and execute multi-step tasks with human-in-the-loop |
| [Prompt Engineering Techniques](notebooks/07-prompt-techniques.ipynb) | Interactive examples of zero-shot, few-shot, chain-of-thought, and their impact on output quality |

---

## How to Use This Handbook

**If you're reading the guides:** Start with Part 1 and work forward. Each guide builds on concepts from the previous one. The guides are designed to be read in 10-15 minutes each.

**If you're running the notebooks:** You'll need Python 3.10+ and the dependencies listed in `requirements.txt`. Each notebook is self-contained with setup instructions at the top. Start with `01-semantic-search-visualized.ipynb` — it's the most visual and gives you an immediate intuition for how the underlying technology works.

### Quick Start

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/genai-engineering-handbook.git
cd genai-engineering-handbook

# Set up a virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Launch Jupyter
jupyter notebook notebooks/
```

### API Keys

Some notebooks require API keys for LLM and embedding model access. Create a `.env` file in the repo root:

```env
OPENAI_API_KEY=your_key_here
# Optional: for Azure OpenAI
AZURE_OPENAI_API_KEY=your_key_here
AZURE_OPENAI_ENDPOINT=your_endpoint_here
```

No API key? Notebooks that require paid APIs will include a free/local alternative where possible (e.g., using open-source embedding models with `sentence-transformers`).

---

## Repo Structure

```
genai-engineering-handbook/
│
├── README.md                          ← You are here
├── requirements.txt                   ← Python dependencies
├── .env.example                       ← Template for API keys
├── .gitignore
│
├── docs/                              ← Written guides (markdown)
│   ├── 01-prompt-engineering-to-context-engineering.md
│   ├── 03-llms-explained.md
│   ├── 04-rag-deep-dive.md
│   ├── 05-rag-vs-long-context.md
│   ├── 06-misconceptions.md
│   ├── 07-agentic-ai.md
│   ├── 08-mcp.md
│   ├── 09-orchestration-patterns.md
│   ├── 10-governance.md
│   ├── 11-mcp-security.md
│   └── 12-context-risks.md
│
├── notebooks/                         ← Jupyter notebooks (hands-on)
│   ├── 01-semantic-search-visualized.ipynb
│   ├── 02-build-a-rag-pipeline.ipynb
│   ├── 03-chunking-strategies.ipynb
│   ├── 04-embeddings-explorer.ipynb
│   ├── 05-rag-vs-long-context.ipynb
│   ├── 06-build-a-simple-agent.ipynb
│   └── 07-prompt-techniques.ipynb
│
└── assets/
    └── images/                        ← Diagrams and visuals
```

---

## Learning Roadmap

```
Week 1: Foundations
├── Read: From Prompt Engineering to Context Engineering
├── Read: LLMs Explained
├── Notebook: Prompt Engineering Techniques
└── Milestone: Can explain role assignment, CoT, few-shot, and context engineering

Week 2: Context & Retrieval
├── Read: RAG Deep Dive
├── Notebook: Semantic Search Visualized
├── Notebook: Embeddings Explorer
└── Milestone: Can explain RAG pipeline and why RAG ≠ training

Week 3: Building RAG Systems
├── Read: RAG vs Long Context
├── Read: Common GenAI Misconceptions
├── Notebook: Build a RAG Pipeline
├── Notebook: Chunking Strategies Compared
└── Milestone: Can build and evaluate a working RAG system

Week 4: Agents & Enterprise
├── Read: Agentic AI Fundamentals
├── Read: MCP + MCP Security
├── Read: AI Governance in Regulated Environments
├── Notebook: Build a Simple Agent
└── Milestone: Can articulate agentic AI architecture and governance needs
```

---

## Key Principles

This handbook is built on a few beliefs:

1. **Understanding beats memorizing.** You don't need to know every API parameter. You need to understand the architecture so you can make the right decisions.

2. **Enterprise reality matters.** Most AI tutorials assume a greenfield startup. This handbook addresses the messy reality: regulated environments, security requirements, legacy systems, and stakeholders who don't speak tech.

3. **Show, don't just tell.** The notebooks aren't exercises for the sake of exercises — they're visualizations and experiments that build intuition you can't get from reading alone.

4. **Misconceptions are as important as concepts.** Knowing what something ISN'T (RAG is not training) is often more valuable than knowing what it is.

---

## Contributing

Contributions are welcome. If you'd like to add a guide, notebook, or correction:

1. Fork the repo
2. Create a branch (`git checkout -b feature/your-addition`)
3. Commit your changes
4. Open a Pull Request with a clear description

Please keep the professional/educational tone consistent with existing content.

---

## About

Created by [Gustavo Marrero](https://linkedin.com/in/gustavoamr) — a software engineering lead with 7+ years in cloud engineering, DevOps, and regulated enterprise environments. Currently leading a team of AI Coaches driving GenAI adoption. This handbook was born from that intersection of hands-on engineering in regulated industries and a passion for making AI accessible — because the best way to learn is to teach.

---

## License

MIT License — use it, share it, build on it.
