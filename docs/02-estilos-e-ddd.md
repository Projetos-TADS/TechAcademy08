# 🏛️ Estilos Arquiteturais e DDD

Detalhamento da modelagem tática e estratégica do sistema sob a ótica do Domain-Driven Design (DDD).

## 1. Context Map (Mapa de Contextos)

O sistema foi decomposto fisicamente seguindo as fronteiras de domínio identificadas.

![Context Map](./assets/context-map.png)

* **Relação de Conformidade:** Ambos os contextos compartilham o mesmo banco de dados (Shared Kernel no nível de persistência), o que gera um acoplamento intencional para simplificar a migração.

---

## 2. Definição de Bounded Contexts

Cada contexto principal foi promovido a um serviço autônomo:

### A. Contexto de Identidade (`auth-service`)
Trata de segurança e gestão de usuários. É a "porta de entrada" lógica para a identificação.
* **Responsabilidades:** Cadastro de usuários (Sign Up) e Emissão de Tokens de Acesso (Sign In/JWT).
* **Módulos no Código:** `User`, `Session`.

### B. Contexto de Catálogo (`catalog-service`)
Engloba o Core Domain (Filmes) e o Subdomínio de Suporte (Favoritos).
* **Responsabilidades:** CRUD de filmes, gestão de elenco (atores/diretores) e listas de favoritos.
* **Módulos no Código:** `Movie`, `Actor`, `Director`, `Favorite`, `MovieImage`.
* *Nota:* O `catalog-service` possui uma representação de leitura da entidade `User` para validar a posse de favoritos.

---

## 3. Entidades e Agregados

Abaixo, a distribuição das entidades principais por serviço.

### No `auth-service`
* **User (Root):** A autoridade máxima sobre os dados do usuário (email, senha, role). É aqui que usuários são criados e autenticados.

### No `catalog-service`
* **Movie (Root):** Agregado principal do catálogo. Controla invariantes como data de lançamento e duração.
    * *Entidades Filhas:* `MovieImage`.
* **Cast & Crew:**
    * **Actor** e **Director** são raízes de seus próprios agregados, permitindo que existam independentemente de filmes.
* **Favorite:**
    * Entidade associativa que liga um `User` (ID) a um `Movie` (ID).
    * *Regra de Negócio:* Só é possível favoritar filmes existentes. A integridade é garantida pelo banco compartilhado.
