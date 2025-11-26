# 🚀 Entrega Contínua (CI/CD)

Esta seção define o pipeline de automação para testes, construção e deploy do sistema Blockbuster.

## 1. Pipeline de CI/CD (Microsserviços)

O pipeline foi configurado para testar e construir cada serviço independentemente, permitindo que falhas no serviço de Catálogo, por exemplo, não impeçam testes no serviço de Autenticação.

### Visualização do Fluxo
O diagrama abaixo ilustra a execução paralela dos jobs para cada microsserviço e para o frontend.

![Pipeline CI/CD](./assets/pipeline-cicd.png)

### Definição do Workflow (GitHub Actions)

O arquivo de configuração encontra-se em `.github/workflows/main.yml`. Ele orquestra os seguintes trabalhos:

1.  **Microservices Checks (Paralelo):**
    * **Auth Service:** Instalação, Build e Testes Unitários.
    * **Catalog Service:** Instalação, Build e Testes Unitários.
    * **Frontend:** Linting e Testes E2E.
2.  **Build & Publish:**
    * Gera 3 imagens Docker distintas: `techacademy-auth`, `techacademy-catalog` e `techacademy-frontend`.
    * Envia para o Docker Hub apenas se todos os testes passarem.
3.  **Deploy:**
    * Conecta via SSH no servidor e atualiza os serviços definidos no `docker-compose.yaml`.

---

## 2. Estratégia de Deploy

Mantemos a estratégia **Recreate (com Downtime Mínimo)** via Docker Compose, que é simples e eficiente para a escala atual.

* **Comportamento:** Ao receber novas imagens, o Docker Compose recria apenas os containers que sofreram alterações.
* **Rollback:** Para reverter, basta alterar a tag da imagem no arquivo `.env` ou `docker-compose.yaml` para a versão anterior e rodar `docker-compose up -d`.

---

## 3. Runbook de Incidentes (Guia de Resposta)

Guia atualizado para depuração dos microsserviços em produção.

### 🚨 Incidente: Erro de Login (Auth Service)
**Sintomas:** Usuário recebe erro 401 ou 500 ao tentar entrar. O Frontend não carrega o token.

1.  **Verificar Logs do Auth Service:**
    ```bash
    docker logs auth-service --tail 100
    ```
2.  **Ação Típica:** Verificar conexão com o banco ou expiração de chaves JWT.

### 🚨 Incidente: Catálogo Vazio ou Lento (Catalog Service)
**Sintomas:** Login funciona, mas a lista de filmes não carrega.

1.  **Verificar Logs do Catalog Service:**
    ```bash
    docker logs catalog-service --tail 100
    ```
2.  **Verificar Cache (Redis):**
    * Se o Redis cair, o catálogo pode ficar lento (buscando direto no MySQL).
    * Check: `docker logs redis_cache`

### 🚨 Incidente: Erro Geral de Conexão (Gateway/Proxy)
**Sintomas:** O site não abre ou dá "Bad Gateway" (502).

1.  **Verificar Nginx:**
    ```bash
    docker logs nginx
    ```
2.  **Verificar se os containers estão de pé:**
    ```bash
    docker-compose ps
    ```
    * *Correção:* `docker-compose up -d --force-recreate <servico_caido>`
