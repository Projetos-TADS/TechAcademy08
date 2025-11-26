# 🛡️ Atributos de Qualidade, Resiliência e Observabilidade

Esta seção define os objetivos de nível de serviço (SLOs), as estratégias de proteção do sistema contra falhas e o plano de monitoramento.

## 1. SLOs e SLIs (Objetivos e Indicadores de Nível de Serviço)

Definimos as metas de confiabilidade para os fluxos críticos (Login e Catálogo).

| Categoria | SLI (Indicador - O que medimos) | SLO (Objetivo - A meta a atingir) | Justificativa |
| :--- | :--- | :--- | :--- |
| **Disponibilidade** | Taxa de requisições HTTP 2xx/3xx vs Total na API. | **99.9%** (aprox. 43 min de downtime/mês) | O catálogo deve estar sempre visível para garantir a conversão de locações. |
| **Latência (Leitura)** | Tempo de resposta do endpoint `GET /movies`. | **95% das req < 200ms** | A navegação pelo catálogo deve ser fluida. Uso de Cache Redis suporta essa meta. |
| **Latência (Escrita)** | Tempo de processamento do endpoint `POST /login`. | **99% das req < 500ms** | O login envolve hash de senha (bcrypt), que é intencionalmente lento, mas não deve travar o usuário. |
| **Taxa de Erro** | Quantidade de respostas HTTP 5xx. | **< 1%** em janelas de 5 minutos | Erros internos indicam bugs graves ou falha de infraestrutura. |

---

## 2. Estratégias de Resiliência Aplicadas

Mecanismos implementados para garantir que o sistema resista a falhas transientes e se recupere automaticamente.

### A. Auto-Healing de Infraestrutura (Container Restart)
* **Estratégia:** O orquestrador (Docker Compose) está configurado com `restart: always`.
* **Comportamento:** Se o processo Node.js encerrar inesperadamente (ex: erro de memória), o container é reiniciado automaticamente.
* **Verificação:** Healthchecks configurados para Database, Redis e Backend garantem que containers travados sejam detectados e reiniciados.

### B. Gestão de Conexões (Database Pooling)
* **Estratégia:** O ORM Sequelize gerencia um *pool* de conexões com o MySQL.
* **Comportamento:** Evita o custo de abrir/fechar conexões a cada requisição e limita o número máximo de conexões simultâneas para não derrubar o banco sob alta carga.

### C. Tratamento Global de Erros (Graceful Error Handling)
* **Estratégia:** Middleware centralizado de erro (`handleError.middleware.ts`) e uso de `express-async-errors`.
* **Comportamento:** Captura exceções não tratadas nas rotas, impedindo que o processo da API quebre (crash) e retornando mensagens amigáveis (JSON) ao cliente em vez de stack traces vazados.

### D. Timeouts (Healthchecks)
* **Estratégia:** Definição de limites de tempo para verificações de saúde.
* **Configuração:** `interval: 10s`, `timeout: 10s`, `retries: 5` no Docker. Evita que o sistema espere indefinidamente por um serviço morto.

---

## 3. Plano de Observabilidade

Estratégia para coleta de Logs e Métricas ("O que está acontecendo agora?").

### 📜 Logs (Logging)
O sistema utiliza a biblioteca **Winston** para logs estruturados e **Morgan** para logs de requisição HTTP.

* **O que coletamos:**
    * **HTTP Logs:** Método, URL, Status Code, Tempo de Resposta (via Morgan).
    * **Application Logs:** Erros de conexão com Banco/Redis, exceções de negócio, inicialização do servidor (via Winston).
* **Formato:** JSON (em produção) para facilitar ingestão por ferramentas como ELK Stack ou Datadog, e Texto simples (em desenvolvimento) para leitura humana.

### 📊 Métricas Sugeridas (Metrics)
Para evolução futura, recomenda-se a instrumentação com **Prometheus** para coletar:

1.  **Node.js Internals:** Uso de CPU, Memória Heap, Event Loop Lag.
2.  **API Throughput:** Requisições por segundo (RPS) agrupadas por Status Code.
3.  **Database Metrics:** Tamanho do Pool de conexões, tempo de execução de queries lentas.
