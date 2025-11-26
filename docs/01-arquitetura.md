# 🏗️ Documentação Arquitetural

## 1. Diagramas C4

### Nível 1: Diagrama de Contexto
Este diagrama situa o sistema no mundo, mostrando quem o utiliza.

![Diagrama de Contexto](./assets/c4-context.png)

### Nível 2: Diagrama de Containers
Este diagrama mostra as aplicações, banco de dados e serviços que compõem o sistema.

![Diagrama de Containers](./assets/c4-container.png)

---

## 2. Registro de Decisões Arquiteturais (ADRs)

### ADR-001: Adoção de Arquitetura Monolítica em Camadas
* **Status:** Aceito.
* **Contexto:** O projeto é uma aplicação de gestão de filmes com requisitos claros de CRUD (Create, Read, Update, Delete) e relacionamentos relacionais fortes (Filmes, Atores, Diretores). A equipe precisa de agilidade no desenvolvimento e simplicidade no deploy.
* **Decisão:** Optou-se por um **Monólito em Camadas (Layered Monolith)**. A estrutura de pastas do backend (`src/controllers`, `src/services`, `src/models`) confirma que o código é organizado por função técnica, e não modularizado por domínio (Modulith) ou separado em microsserviços.
* **Consequências:**
    * (+) **Positivo:** Simplicidade para desenvolver, testar e implantar (apenas um container de backend).
    * (+) **Positivo:** Baixa latência interna, pois as chamadas entre módulos (ex: Usuário chamando Filmes) são chamadas de função em memória, não chamadas de rede.
    * (-) **Negativo:** Pode crescer desordenadamente se as barreiras entre as camadas (Controller -> Service -> Model) não forem respeitadas.

### ADR-002: Estratégia de Cache com Redis
* **Status:** Aceito.
* **Contexto:** Operações de leitura de catálogo (listar filmes) costumam ser muito mais frequentes que escritas. É necessário garantir alta performance na listagem.
* **Decisão:** Utilização do **Redis** como camada de cache.
* **Consequências:**
    * (+) **Positivo:** Reduz a carga no banco de dados MySQL para consultas repetitivas.
    * (+) **Positivo:** Melhora o tempo de resposta para o usuário final.
    * (-) **Negativo:** Adiciona complexidade de infraestrutura (mais um container para gerenciar) e necessidade de estratégia de invalidação de cache.

---

## 3. Cenários de Qualidade

### 🟢 Disponibilidade
* **Cenário:** O container de banco de dados ou backend pode falhar inesperadamente.
* **Estratégia:** Uso da diretiva `restart: always` e configuração de `healthcheck` no `docker-compose.yaml` para garantir que o orquestrador (Docker) reinicie serviços travados automaticamente.

### 🚀 Desempenho
* **Cenário:** Listagem de filmes em horários de pico.
* **Estratégia:** Implementação de cache distribuído via Redis (`ioredis` no `package.json`) para servir dados frequentes em latência de sub-milissegundos, evitando round-trip ao disco do MySQL.

### 🛠️ Manutenibilidade
* **Cenário:** Novos desenvolvedores entrando no projeto.
* **Estratégia:** Uso estrito de **TypeScript** e **ESLint** (presentes nas dependências) para garantir tipagem estática e padronização de código, reduzindo erros em tempo de execução.
