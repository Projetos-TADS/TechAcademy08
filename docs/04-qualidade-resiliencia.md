# 🛡️ Atributos de Qualidade, Resiliência e Observabilidade

Esta seção define os objetivos de nível de serviço (SLOs) para cada microsserviço e as estratégias de isolamento de falhas implementadas.

## 1. SLOs e SLIs (Por Serviço)

Definimos metas específicas para os contextos de Autenticação e Catálogo.

| Serviço | Métrica (SLI) | Meta (SLO) | Justificativa |
| :--- | :--- | :--- | :--- |
| **Auth Service** | Latência de Login (`POST /login`) | **99% < 500ms** | O hashing de senha é custoso, mas o login deve ser rápido o suficiente para não frustrar o usuário. |
| **Auth Service** | Disponibilidade | **99.95%** | O login é a porta de entrada. Se cair, ninguém entra no sistema. |
| **Catalog Service** | Latência de Listagem (`GET /movies`) | **95% < 200ms** | A navegação deve ser instantânea. O cache Redis suporta essa meta agressiva. |
| **Catalog Service** | Disponibilidade | **99.9%** | Falhas aqui são toleráveis (usuário logado pode ver outras telas), mas impactam a conversão. |

---

## 2. Estratégias de Resiliência Aplicadas

### A. Isolamento de Falhas (Bulkhead Pattern)
* **Estratégia:** Separação física dos processos em containers distintos.
* **Comportamento:** Uma falha crítica de memória no `catalog-service` não derruba o `auth-service`. O usuário consegue fazer login e ver seu perfil, mesmo que a lista de filmes esteja temporariamente indisponível.

### B. Auto-Healing Distribuído
* **Estratégia:** Orquestração de reinício automático via Docker.
* **Comportamento:** Cada serviço possui seu próprio `healthcheck`. Se o `auth-service` parar de responder na porta 3001, apenas ele será reiniciado, sem interromper o Catálogo.

### C. Proteção do Banco de Dados (Connection Pooling)
* **Estratégia:** Limite de conexões por serviço.
* **Implementação:** Como o banco é compartilhado, cada microsserviço gerencia seu próprio *pool* de conexões via Sequelize, garantindo que um serviço não consuma todas as conexões disponíveis do MySQL, deixando o outro sem acesso.

---

## 3. Plano de Observabilidade (Distribuída)

Com microsserviços, "olhar os logs" tornou-se mais complexo. A estratégia baseia-se na segregação de logs por responsabilidade.

### 📜 Logs por Container
Utilizamos a biblioteca **Winston** em cada serviço para padronizar o formato JSON.

1.  **Logs de Acesso e Segurança (`auth-service`):**
    * Monitorar tentativas de login falhas e criação de usuários.
    * Comando: `docker logs auth-service -f`
2.  **Logs de Negócio (`catalog-service`):**
    * Monitorar erros de cache (Redis) e queries lentas de filmes.
    * Comando: `docker logs catalog-service -f`
3.  **Logs de Tráfego (`nginx`):**
    * Visão unificada de todas as requisições que chegam ao sistema (Status 200, 404, 500).
    * Comando: `docker logs nginx -f`

### 📊 Métricas de Infraestrutura
Monitoramento básico via Docker Stats para garantir que a divisão de serviços não estoure os recursos do servidor:
* **CPU/Memória por Container:** Verificar se o `catalog-service` (que processa listas grandes) consome mais memória que o `auth-service`.
