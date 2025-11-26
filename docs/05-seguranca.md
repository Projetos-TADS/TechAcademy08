# 🔒 Segurança e DevSecOps

Esta seção descreve as estratégias de defesa do sistema, controle de acesso e práticas de segurança no ciclo de desenvolvimento.

## 1. Threat Model (Modelagem de Ameaças)

Focamos na análise do fluxo crítico de **Autenticação (Login)**, utilizando uma abordagem simplificada do método STRIDE.

| Componente | Ameaça (O que pode dar errado?) | Classificação (STRIDE) | Mitigação Implementada (Defesa) |
| :--- | :--- | :--- | :--- |
| **API Endpoint** (`POST /login`) | Atacante tenta adivinhar senhas (Brute Force). | **S**poofing | Hashing de senha com `bcryptjs` (lento por design) torna ataques de força bruta computacionalmente caros. |
| **API Endpoint** (`POST /login`) | Injeção de comandos SQL maliciosos no campo de email. | **T**ampering | Uso do ORM **Sequelize**, que utiliza *Prepared Statements* nativamente, sanitizando inputs. |
| **Tráfego de Rede** | Interceptação de credenciais ou Tokens em redes Wi-Fi públicas. | **I**nformation Disclosure | O sistema obriga tráfego criptografado (HTTPS) através do Proxy Reverso Nginx (porta 443). |
| **Banco de Dados** | Vazamento de senhas em caso de dump do banco. | **I**nformation Disclosure | As senhas nunca são salvas em texto plano; apenas o hash é persistido. |
| **Frontend** | Cross-Site Scripting (XSS) injetando scripts em descrições de filmes. | **T**ampering | O React (Frontend) escapa automaticamente o conteúdo renderizado no DOM, prevenindo a execução de scripts arbitrários. |

---

## 2. Estratégia de Autenticação e Autorização

O sistema utiliza uma estratégia **Stateless** baseada em Tokens, adequada para a arquitetura REST.

### Autenticação (Quem é você?)
* **Mecanismo:** JWT (JSON Web Token).
* **Fluxo:** O usuário envia credenciais `(email, password)`. Se válidas, o backend assina um token contendo o `sub` (ID do usuário) e `role` (papel).
* **Segurança do Token:** O token é assinado com uma `SECRET_KEY` segura (definida via variáveis de ambiente) e possui tempo de expiração (`EXPIRES_IN`).

### Autorização (O que você pode fazer?)
Implementação de **RBAC (Role-Based Access Control)** com dois níveis de privilégio:

1.  **ADMIN:** Acesso total, incluindo operações de escrita no catálogo (Criar/Editar Filmes, Atores).
    * *Middleware:* `isAdmin.middleware.ts`.
2.  **USER:** Acesso de leitura ao catálogo e gestão de recursos próprios (Perfil, Favoritos).
    * *Middleware:* `verifyToken.middleware.ts` para rotas protegidas gerais.

---

## 3. Checklist de Segurança no Pipeline (DevSecOps)

Práticas de segurança "Shift-Left" aplicadas automaticamente antes do deploy.

### ✅ Análise Estática de Código (SAST)
* **Ferramenta:** ESLint.
* **Onde roda:** No hook de `pre-commit` (via Husky) e no CI.
* **Função:** Detecta padrões de código inseguros, variáveis não utilizadas e violações de boas práticas antes que o código entre no repositório.

### ✅ Validação de Dados (Input Validation)
* **Ferramenta:** Zod.
* **Onde roda:** Em tempo de execução (Runtime) na entrada da API.
* **Função:** Garante que os dados recebidos (formato de email, tamanho de senha, tipos de dados) estejam estritamente dentro do esperado, rejeitando payloads maliciosos ou malformados.

### ✅ Gestão de Segredos
* **Estratégia:** `.env`.
* **Implementação:** Arquivos `.env` contendo senhas de banco e chaves JWT são listados no `.gitignore` e nunca versionados. Em produção, estas variáveis são injetadas pelo orquestrador de containers.
