# 🔌 Integração, APIs e Dados

Esta seção documenta a interface de comunicação dos microsserviços e a estratégia de dados compartilhada.

## 1. Especificação da API (OpenAPI/Swagger)

Com a arquitetura distribuída, cada serviço é responsável por documentar seus próprios endpoints. O API Gateway (Nginx) unifica o acesso para o cliente.

* **Estratégia:** Code-First (Gerado via JSDoc/TS em cada serviço).
* **Endpoints de Documentação (Dev):**
    * **Auth Service:** `http://localhost:3001/v1/docs` (Gestão de Identidade)
    * **Catalog Service:** `http://localhost:3002/v1/docs` (Catálogo e Operações)

### Mapeamento de Rotas por Serviço
O Nginx roteia as requisições baseando-se no prefixo da URL:

| Serviço | Prefixo Rota | Principais Recursos |
| :--- | :--- | :--- |
| **Auth Service** | `/v1/login`, `/v1/users` | Autenticação (JWT), Cadastro de Usuários. |
| **Catalog Service** | `/v1/movies`, `/v1/actors` | Listagem de Filmes, Detalhes, Atores, Diretores. |
| **Catalog Service** | `/v1/favorites` | Adicionar/Remover favoritos (Valida usuário via Token). |

---

## 2. Modelagem de Dados

O diagrama abaixo representa o modelo lógico do banco de dados. Embora o acesso seja feito por serviços diferentes, as tabelas residem no mesmo esquema físico (MySQL).

![Modelo Entidade Relacionamento](./assets/er-diagram.png)

* **Propriedade dos Dados:**
    * A tabela `Users` é escrita exclusivamente pelo **Auth Service**.
    * As tabelas `Movies`, `Actors`, `Directors` são gerenciadas pelo **Catalog Service**.
    * A integridade referencial (FKs) é mantida pelo motor do banco de dados.

---

## 3. Registro de Decisões Arquiteturais (ADRs) - Dados

### ADR-003: Banco de Dados Compartilhado (Shared Database)

* **Status:** Aceito (Transicional).
* **Contexto:** Ao migrar de um monólito para microsserviços, a separação imediata dos dados é a tarefa mais arriscada e complexa. O negócio exige a evolução rápida do Catálogo, mas não quer reescrever a camada de dados inteira agora.
* **Decisão:** Manter uma única instância **MySQL** compartilhada entre `auth-service` e `catalog-service`.
* **Consequências:**
    * (+) **Consistência Forte:** Continuamos usando Foreign Keys rígidas (ex: Não é possível favoritar um filme que não existe no DB), o que evita inconsistência de dados durante a migração.
    * (+) **Facilidade Operacional:** Apenas um container de banco para backup e monitoramento.
    * (-) **Acoplamento:** Uma mudança de schema na tabela `Users` (feita pelo time de Auth) pode quebrar queries do `catalog-service` se não for comunicada.
    * (-) **Single Point of Failure:** Se o MySQL cair, todos os microsserviços param.

### ADR-004: Cache Centralizado (Redis)
* **Status:** Aceito.
* **Decisão:** Uso de uma instância Redis dedicada para cache de respostas HTTP do Catálogo.
* **Justificativa:** Reduzir a latência de leitura em 95% das requisições públicas (listagem de filmes).
