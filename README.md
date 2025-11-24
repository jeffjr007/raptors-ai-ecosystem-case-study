# 🦖🏀 Raptors AI Ecosystem - Case Study de Automação Inteligente

> ⚠️ **Nota de Confidencialidade:** Este repositório documenta a arquitetura técnica e as decisões de engenharia do projeto "Raptors AI". O código-fonte completo e os dados sensíveis não foram disponibilizados publicamente para proteger a propriedade intelectual e a privacidade do cliente (LGPD).

## 🎯 Visão Geral
Este projeto é um ecossistema de automação "End-to-End" desenvolvido para modernizar a operação da **Escolinha de Basquete Raptors**.

O objetivo foi eliminar gargalos administrativos, transformando processos manuais (matrículas em papel, conferência de PIX visual, cobrança manual) em um sistema autônomo, escalável e integrado ao WhatsApp, que é o canal preferido dos clientes.

## 🚩 O Desafio
A escola enfrentava três problemas críticos que limitavam seu crescimento:
1.  **Sobrecarga Operacional:** Os professores perdiam horas respondendo dúvidas repetitivas e realizando matrículas manuais no WhatsApp.
2.  **Gestão Financeira Frágil:** A conferência de pagamentos dependia de olhar prints de comprovantes e dar baixa manual em planilhas, gerando erros e inadimplência.
3.  **Dados Descentralizados:** As informações dos alunos ficavam espalhadas entre conversas de chat e planilhas desatualizadas.

## 💡 A Solução Engenheirada
Diferente de chatbots lineares simples, desenvolvi uma arquitetura baseada em **Agentes Especializados** e **Filas de Processamento**, orquestrada via n8n e com persistência de dados em PostgreSQL.

O sistema opera em três pilares principais:

### 1. Módulo Receptivo (Atendimento & Matrícula)
* **Router Agent (O "Cérebro"):** Utilizei um LLM (OpenAI) para classificar a intenção do usuário. O sistema não segue uma árvore de decisão fixa; ele entende o contexto e roteia para o agente especialista (Matrícula, Financeiro ou Dúvidas).
* **Auditoria Visual (Vision AI):** Implementei uma funcionalidade onde o usuário envia a foto/PDF do comprovante PIX. O sistema usa **GPT-4o (Vision)** para ler a imagem, extrair os dados (pagador, valor, data), validar contra o valor esperado e dar baixa automática no banco de dados.
* **Fila de Mensagens (Queue):** Para lidar com picos de mensagens sem perder dados, criei uma tabela de fila no PostgreSQL. O n8n processa essa fila garantindo a ordem de chegada e atomicidade das transações.

### 2. Módulo Ativo (Cobrança & Dunning)
* **Cobrança Inteligente:** Um *Cron Job* dispara diariamente, executa uma **RPC (Stored Procedure)** no banco de dados para identificar alunos inadimplentes e calcula juros/valores dinamicamente baseados no perfil do aluno (Sócio vs. Não Sócio).
* **Anti-Spam & Rate Limiting:** Para proteger o número do WhatsApp contra bloqueios, implementei um algoritmo de **Jitter**. O sistema envia as cobranças em lotes pequenos com atrasos aleatórios (20s a 40s) entre cada mensagem, simulando comportamento humano.

### 3. Sincronização de Dados (ETL)
* **Pipeline Database-to-Sheets:** Para manter a familiaridade do cliente com planilhas, criei um pipeline ETL que extrai a "verdade" do banco de dados (Supabase) e atualiza o Google Sheets diariamente. Isso garante que a interface operacional (planilha) esteja sempre sincronizada com o sistema transacional.

## 🛠️ Stack Tecnológica

| Categoria | Tecnologias |
| :--- | :--- |
| **Orquestração** | n8n (Self-hosted), Webhooks |
| **Inteligência Artificial** | OpenAI (GPT-4o Mini para texto, GPT-4o para visão), LangChain |
| **Banco de Dados** | Supabase (PostgreSQL), Stored Procedures (PL/pgSQL) |
| **Canais** | Evolution API (WhatsApp Gateway), Chatwoot (Human Handoff) |
| **Integração** | Google Sheets API, REST APIs |

---

## 📐 Arquitetura do Sistema

O diagrama abaixo ilustra o fluxo de dados, a interação entre os agentes e a lógica de processamento assíncrono implementada no projeto.

```mermaid
graph TD
    %% Definição de Estilos (Texto forçado em Preto)
    classDef user fill:#f9f,stroke:#333,stroke-width:2px,color:#000;
    classDef ai fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000;
    classDef db fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#000;
    classDef batch fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#000;

    User([📱 Usuário/WhatsApp]):::user <-->|Mensagens| Evo[Evolution API]
    Evo <-->|Webhook| N8N_Queue{⚡ Fila de Mensagens<br/>PostgreSQL}

    subgraph "Módulo Receptivo (n8n)"
        N8N_Queue -->|Processar Item| Router[🤖 Router Agent / Classificador]:::ai
        
        Router -->|Intenção: Matrícula| Ag_Matricula[📝 Agente de Matrícula]:::ai
        Router -->|Intenção: Dúvida| Ag_FAQ[💬 Agente de Dúvidas]:::ai
        Router -->|Intenção: Pagamento| Ag_Finan[💸 Agente Financeiro]:::ai
        Router -->|Anexo: Comprovante| Visao[👁️ Vision AI / Auditoria]:::ai
        
        Visao -->|Extração de Dados| GPT4o[🧠 GPT-4o Vision]:::ai
        GPT4o -->|Validar & Baixar| DB_Pagamentos
    end

    subgraph "Camada de Dados (Supabase)"
        Ag_Matricula -->|Insert/Update| DB_Alunos[(🗄️ Tabela Alunos)]:::db
        Ag_Finan -->|Select Status| DB_Pagamentos[(🗄️ Tabela Pagamentos)]:::db
        
        DB_Alunos -.->|RPC: Query Inadimplentes| Batch_Process
    end

    subgraph "Módulo Ativo & ETL (Background)"
        Cron[⏰ Cron Job] --> Batch_Process[⚡ Cobrança Ativa]:::batch
        Batch_Process -->|Jitter / Delay Aleatório| Evo
        
        Cron_ETL[⏰ Cron Madrugada] --> ETL[🔄 Pipeline ETL]:::batch
        ETL -->|Select| DB_Alunos
        ETL -->|Sync/Update| GSheets[📊 Google Sheets Professor]:::user
    end

    Router -->|Transbordo/Erro| Humano[👨‍💻 Chatwoot / Humano]:::user
````

## 🚀 Resultados Obtidos

  * **100% de Automação:** Eliminação total da necessidade de digitação manual de matrículas.
  * **Redução de Inadimplência:** O sistema de cobrança ativa garantiu lembretes constantes sem desgaste da equipe.
  * **Segurança de Dados:** Centralização das informações em banco SQL robusto, eliminando perda de dados em chats.

-----

*Desenvolvido por [Jeferson Junior](https://www.linkedin.com/in/jeferson-junior-as/)*

```
```
