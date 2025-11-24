## 📐 Arquitetura do Sistema

```mermaid
graph TD
    User([📱 Usuário/WhatsApp]) -->|Mensagem| Evo[Evolution API]
    Evo -->|Webhook| N8N{⚡ n8n Workflow}
    
    subgraph "Camada de Processamento (n8n)"
        N8N -->|Sanitização| Router[🤖 Router Agent / OpenAI]
        
        Router -->|Intenção: Matrícula| Matricula[📝 Agente de Matrícula]
        Router -->|Intenção: Financeiro| Financeiro[💸 Agente Financeiro]
        Router -->|Intenção: Comprovante| Vision[👁️ Vision AI / Auditoria]
        Router -->|Erro/Complexo| Humano[👨‍💻 Escalar para Humano]
    end
    
    subgraph "Banco de Dados & Ferramentas"
        Matricula <-->|CRUD Alunos| Supabase[(🗄️ Supabase / PostgreSQL)]
        Financeiro <-->|Verificar Status| Supabase
        Vision -->|Baixa Automática| Supabase
        Financeiro -->|Gerar PIX| Code[💻 JS Function]
    end
    
    subgraph "Cobrança Ativa (Batch)"
        Cron[⏰ Cron Job Diário] -->|RPC Call| Supabase
        Supabase -->|Lista Inadimplentes| N8N_Batch[⚡ n8n Dunning Workflow]
        N8N_Batch -->|Jitter/Delay| Evo
    end
    
    Humano --> Chatwoot[💬 Chatwoot Dashboard]
