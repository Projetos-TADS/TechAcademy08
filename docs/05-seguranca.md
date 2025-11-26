# 🔒 Segurança e DevSecOps

Esta seção descreve as estratégias de defesa do sistema em profundidade, controle de acesso e práticas de segurança no ciclo de vida de desenvolvimento.

## 1. Threat Model (Modelagem de Ameaças)

A superfície de ataque expandiu-se com a exposição de múltiplos serviços. Focamos a análise nas fronteiras de comunicação.

| Componente | Ameaça (O que pode correr mal?) | Classificação (STRIDE) | Mitigação Implementada |
| :--- | :--- | :--- | :--- |
| **API Gateway (Nginx)** | Ataques de negação de serviço (DDoS) ou bypass de rotas. | **E**levation of Privilege | O Nginx atua como único ponto de entrada público (Porta 443), ocultando a topologia interna dos serviços. |
| **Auth Service** | Roubo de tokens de acesso (JWT). | **I**nformation Disclosure | Tokens têm tempo de vida curto (`EXPIRES_IN`) e são assinados com um segredo robusto (`SECRET_KEY`). |
| **Catalog Service** | Acesso indevido a funcionalidades administrativas. | **T**ampering | Middleware de verificação valida a assinatura do JWT localmente antes de processar qualquer pedido de escrita. |
| **Shared Database** | Um serviço comprometer dados do outro. | **T**ampering | Embora partilhem a instância física, o acesso é logicamente segregado pelo código da aplicação (o `catalog-service` não possui *Models* para escrita de Users, apenas leitura). |
| **Rede Interna** | Sniffing de tráfego entre serviços. | **I**nformation Disclosure | Os serviços comunicam através de uma rede Docker privada (`app-network`), inacessível à internet pública. |

---

## 2. Estratégia de Autenticação e Autorização

Adotamos um padrão de **Identity Provider (IdP)** descentralizado.

### Autenticação (Emissão)
* **Responsável:** `auth-service`.
* **Mecanismo:** Ao receber credenciais válidas (`email/password`), este serviço gera um **JWT (JSON Web Token)** assinado contendo o `sub` (ID do utilizador) e a `role`.
* **Segurança:** O hashing da palavra-passe é feito com `bcryptjs` exclusivamente neste serviço.

### Autorização (Validação)
* **Responsável:** `catalog-service` (e outros futuros serviços).
* **Mecanismo:** Validação *Stateless*. O serviço de catálogo partilha a mesma `SECRET_KEY` (variável de ambiente) que o serviço de autenticação.
* **Fluxo:**
    1.  O Nginx encaminha o pedido com o cabeçalho `Authorization: Bearer <token>`.
    2.  O `catalog-service` verifica a assinatura do token sem precisar de contactar o `auth-service` (ganho de desempenho).
    3.  Se válido, o ID do utilizador é extraído para o contexto do pedido.

---

## 3. Checklist de Segurança no Pipeline (DevSecOps)

A automação de segurança foi replicada para cobrir todos os repositórios de serviço.

### ✅ Análise Estática (SAST)
* **Configuração:** O ESLint corre independentemente nas pastas `services/auth-service`, `services/catalog-service` e `frontend`.
* **Objetivo:** Garantir que nenhum código com vulnerabilidades conhecidas (ex: uso de `eval()`) chegue ao repositório.

### ✅ Validação de Inputs (Runtime)
* **Ferramenta:** Zod.
* **Aplicação:** Todos os DTOs (Data Transfer Objects) de entrada são validados.
    * No `auth-service`: Validação rigorosa de formato de email e complexidade de palavra-passe.
    * No `catalog-service`: Validação de tipos de dados para filmes e atores.

### ✅ Gestão de Segredos
* **Segregação:** Cada serviço possui o seu próprio ficheiro `.env` em desenvolvimento, mas em produção (CI/CD), as variáveis sensíveis são injetadas como *Secrets* do GitHub Actions, nunca hardcoded no Dockerfile.
