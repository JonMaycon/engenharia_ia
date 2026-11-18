# Engenharia de IA

Repositório prático de uma formação em Engenharia de IA. Reúne notebooks de fundamentos e quatro projetos incrementais que vão de um agente com ferramentas até o deploy em nuvem com infraestrutura como código.

**Material do curso:** [jonmaycon.github.io/engenharia_ia](https://jonmaycon.github.io/engenharia_ia/). É a página interativa com os 15 módulos, as etapas de cada um e o passo a passo dos laboratórios baseados neste código. O código-fonte da página está em [`docs/index.html`](docs/index.html).

## Visão geral

| Pasta | O que implementa | Principais tecnologias | Módulos do curso |
|---|---|---|---|
| [`00-tutorials`](00-tutorials) | Notebooks de fundamentos: chamadas a LLMs, agentes, RAG e técnicas de prompt | LangChain, Groq, Gemini, OpenAI, FAISS, Tavily | 1, 2, 3, 5 |
| [`01-simple-react-agent`](01-simple-react-agent) | Agente ReAct que consulta e altera um banco de produtos, estoque e vendas | LangChain `create_agent`, Groq, PostgreSQL, FastAPI, Chainlit | 2, 8 |
| [`02-agentic-rag`](02-agentic-rag) | RAG agêntico sobre transcrições de vídeos, com pipeline de ingestão | Pinecone, embeddings Google, Gemini, Groq, SQLite | 5 |
| [`03-multiagent`](03-multiagent) | Fluxo em grafo com verificação de segurança, reescrita da pergunta, memória curta e streaming | LangGraph, `PostgresSaver`, PostgreSQL, FastAPI | 4, 7 |
| [`04-ecs-aws`](04-ecs-aws) | Backend e frontend em contêineres implantados na AWS por pipeline | Docker, SSE, Terraform, ECR, ECS Fargate, ALB, Secrets Manager | 8, 9 |

Os projetos aumentam de complexidade em sequência: ferramentas e laço de agente → recuperação de conhecimento → orquestração com estado persistido → empacotamento e operação em nuvem.

## Arquitetura dos projetos

### 01-simple-react-agent

```
Chainlit (:8001) ──POST /chat──▶ FastAPI (:8000) ──▶ Agente LangChain (Groq)
                                                       ├─ read_database_metadata ──▶ database_metadata.txt
                                                       └─ execute_sql ─────────────▶ PostgreSQL (:5432)
```

O agente lê a descrição do esquema antes de gerar SQL e recebe os erros do banco como observação, o que permite que ele corrija a consulta e tente de novo.

### 02-agentic-rag

```
Ingestão (offline)
  data/*.json ──▶ resumo (Groq) ──▶ embeddings (Google) ──▶ Pinecone: pod-academy (resumos)
                                                       └──▶ Pinecone: titles-videos (títulos)
              └──▶ transcrição completa ──▶ SQLite (videos.db)

Consulta (online)
  Chainlit ──▶ FastAPI ──▶ Agente (Gemini)
                            ├─ search_video_summaries ──▶ Pinecone
                            ├─ search_video_titles ─────▶ Pinecone
                            └─ get_video_transcript ────▶ SQLite
```

A busca semântica acontece sobre resumos curtos. A resposta usa a transcrição completa do vídeo encontrado.

### 03-multiagent

```
Chainlit (thread_id) ──▶ FastAPI (streaming) ──▶ LangGraph
    check_security ──(inseguro)──▶ END
          │
          ▼
    improve_question ──▶ main_llm ──▶ tokens em streaming
                            ├─ lê/grava a tabela "memory-short"
                            └─ checkpoints do grafo (PostgresSaver)
```

O roteamento é feito por `Command(goto=...)` dentro de cada nó. O histórico curto (as 10 últimas mensagens por `thread_id`) e os checkpoints do grafo ficam no PostgreSQL.

### 04-ecs-aws

```
git push ──▶ GitHub Actions
               ├─ Terraform (terraform_ecr) ──▶ repositórios ECR
               ├─ docker build/push (tag = SHA do commit)
               └─ Terraform (terraform) ──▶ ECS Fargate + ALB + IAM + Secrets Manager + CloudWatch

Navegador ──▶ ALB ──▶ Chainlit (:8005) ──▶ ALB ──▶ FastAPI (:8000, SSE) ──▶ Groq
                                                      └─ chave lida do Secrets Manager
```

## Estrutura do repositório

```
.
├── 00-tutorials/              # Notebooks 01 a 04 e dataset de exemplo (03 - Pessoas.json)
├── 01-simple-react-agent/
│   ├── backend/               # agent.py, tools.py, prompts.py, main.py, database_metadata.txt
│   ├── database/              # docker-compose.yaml, create_tables.sql
│   ├── frontend/              # app.py (Chainlit), env.example
│   └── requirements.txt
├── 02-agentic-rag/
│   ├── backend/               # agent.py, tools.py, prompts.py, main.py, test_ollama.py
│   ├── data-engineering/      # ingest_pipeline.py, test.py, data/*.json, videos.db
│   ├── frontend/
│   └── requirements.txt
├── 03-multiagent/
│   ├── backend/               # agent.py (grafo), memory_short.py, main.py
│   ├── database/              # docker-compose.yaml, create_memory_short.sql
│   ├── frontend/
│   └── requirements.txt
├── 04-ecs-aws/
│   ├── backend/               # app.py (FastAPI + SSE), Dockerfile
│   ├── frontend/              # app.py (Chainlit), Dockerfile
│   ├── terraform_ecr/         # repositórios de imagens
│   └── terraform/             # cluster, serviços, ALBs, IAM, segredos, logs, autoscaling
├── .github/workflows/         # 04-ecs-aws.yml
└── docs/                      # página do curso (GitHub Pages)
```

## Pré-requisitos

- Python 3.10 ou superior
- Docker com Docker Compose (projetos 01, 03 e 04)
- Contas e chaves de API, conforme o projeto:

| Serviço | Uso | Projetos |
|---|---|---|
| [Groq](https://console.groq.com) | LLM principal | 01, 02 (ingestão), 03, 04 |
| [Google AI Studio](https://aistudio.google.com) | Gemini e embeddings | 02 |
| [Pinecone](https://app.pinecone.io) | Banco vetorial | 02 |
| AWS | Infraestrutura | 04 |
| Ollama (opcional) | Execução local de modelos | 02 |

## Início rápido

Exemplo com o projeto `01-simple-react-agent` no Windows (PowerShell). No Linux e no macOS, ative o ambiente com `source myenv/bin/activate`.

```powershell
# 1. Ambiente virtual único na raiz, compartilhado pelos projetos
python -m venv myenv
.\myenv\Scripts\Activate.ps1
pip install -r 01-simple-react-agent\requirements.txt

# 2. Banco de dados
cd 01-simple-react-agent
docker compose -f database/docker-compose.yaml up -d
Get-Content database/create_tables.sql | docker exec -i simple-react-agent-postgres psql -U app_user -d app_db

# 3. Credenciais (preencha GROQ_API_KEY e POSTGRES_*)
New-Item backend\.env -ItemType File
Copy-Item frontend\env.example frontend\.env

# 4. Terminal 1: API
cd backend
uvicorn main:app --reload --host 127.0.0.1 --port 8000

# 5. Terminal 2: interface (ative o myenv antes)
cd 01-simple-react-agent\frontend
chainlit run app.py -w --port 8001
```

- API e documentação interativa: http://127.0.0.1:8000/docs
- Chat: http://localhost:8001

Guias completos de ambiente estão em [`01-simple-react-agent/GUIA_AMBIENTE_WINDOWS.md`](01-simple-react-agent/GUIA_AMBIENTE_WINDOWS.md) e [`GUIA_AMBIENTE_LINUX.md`](01-simple-react-agent/GUIA_AMBIENTE_LINUX.md). O passo a passo de cada projeto está nos laboratórios da [página do curso](https://jonmaycon.github.io/engenharia_ia/).

## Configuração

Cada backend carrega as variáveis de `backend/.env`, que deve ser criado a partir do `env.example` correspondente. O `.gitignore` impede que arquivos `.env` sejam versionados.

| Variável | Projetos | Descrição |
|---|---|---|
| `GROQ_API_KEY` | 01, 02, 03 | Chave do Groq |
| `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | 01, 03 | Conexão com o PostgreSQL |
| `DATABASE_URL` | 01, 03 | Alternativa às variáveis `POSTGRES_*`, no formato `postgresql://usuario:senha@host:5432/banco` |
| `CHECKPOINT_DATABASE_URL` | 03 | Banco dos checkpoints do LangGraph; usa `DATABASE_URL` se ausente |
| `USE_MEMORY_CHECKPOINTER` | 03 | `1` para guardar checkpoints em memória, sem PostgreSQL |
| `GOOGLE_API_KEY` | 02 | Chave do Google AI Studio |
| `PINECONE_API_KEY` | 02 | Chave do Pinecone |
| `GOOGLE_GENAI_MODEL`, `GOOGLE_GENAI_TEMPERATURE` | 02 | Modelo de chat Gemini e temperatura |
| `GOOGLE_EMBEDDING_MODEL` | 02 | Modelo de embeddings; precisa ser o mesmo na ingestão e na consulta |
| `GROQ_MODEL` | 02 | Modelo usado para gerar os resumos na ingestão |
| `CHAT_API_BASE_URL` | frontends 01, 02, 03 | Endereço da API |
| `BACKEND_MESSAGE_URL` | frontend 04 | URL do endpoint `POST /message` |
| `GROQ_SECRET_ID` | backend 04 | Nome do segredo no AWS Secrets Manager |

No projeto 04 o backend não usa `.env`: a chave do Groq é lida do AWS Secrets Manager em tempo de execução.

## CI/CD

O workflow [`.github/workflows/04-ecs-aws.yml`](.github/workflows/04-ecs-aws.yml) roda em push para qualquer branch quando há mudanças em `04-ecs-aws/` ou no próprio workflow, e também pode ser disparado manualmente.

- `main` implanta no ambiente `prd`; as demais branches, em `dev`.
- O estado do Terraform fica em um bucket S3, com uma chave por ambiente.
- As imagens recebem como tag o SHA do commit, o que força uma nova task definition a cada deploy.

Segredos necessários no repositório (Settings > Secrets and variables > Actions): `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` e `GROQ_API_KEY`.

Antes de usar em outra conta, ajuste no workflow `ECR_REGISTRY` (número da conta AWS) e `TF_STATE_BUCKET` (bucket do estado, que precisa existir antes). O deploy cria recursos pagos, como Application Load Balancers e tarefas Fargate; destrua-os com `terraform destroy` após o uso.

## Limitações conhecidas

O código tem finalidade didática. Pontos a tratar antes de qualquer uso em produção:

- **Modelo de embeddings descontinuado.** `02-agentic-rag` usa `models/text-embedding-004`, desligado na API Gemini em janeiro de 2026. Use um modelo atual (por exemplo, `models/gemini-embedding-001`) e recrie os índices do Pinecone com a dimensão do novo modelo.
- **Nomes de modelos.** Os modelos padrão do Groq e do Gemini citados no código podem ser descontinuados. Confira as listas atuais dos provedores.
- **Execução de SQL sem restrição.** A ferramenta `execute_sql` do projeto 01 executa qualquer comando, inclusive `DELETE` e `DROP`, com autocommit.
- **APIs sem autenticação.** Nenhum backend exige credencial do cliente, e os load balancers do projeto 04 usam apenas HTTP.
- **Credenciais de longa duração no CI.** O pipeline usa chaves de acesso da AWS; o recomendado é federação OIDC entre GitHub e AWS.
- **Chave no estado do Terraform.** A chave do Groq passa por variável do Terraform e fica gravada no arquivo de estado. Proteja o bucket com criptografia e acesso restrito.
- **Dependências sem versão fixa.** Os arquivos `requirements.txt` não fixam versões, então builds em datas diferentes podem gerar ambientes diferentes.
- **Porta compartilhada.** Os projetos 01 e 03 usam a porta 5432; rode um banco por vez.
- **`.gitignore` incompleto.** Não cobre arquivos do Terraform (`.terraform/`, `*.tfstate`, `terraform.tfvars`).
