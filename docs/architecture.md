# 📐 Arquitetura de Solução — Diagramas

---

## Índice

1. [Fluxo da Jornada do Usuário](#1-fluxo-da-jornada-do-usuário)
2. [Arquitetura de Microsserviços](#2-arquitetura-de-microsserviços)
3. [Diagrama de Sequência](#3-diagrama-de-sequência)
4. [Strategy Pattern — Elegibilidade](#4-strategy-pattern--elegibilidade)
5. [Chain of Responsibility — Validação](#5-chain-of-responsibility--validação)
6. [Modelo de Dados](#6-modelo-de-dados)
7. [Estados da Proposta](#7-estados-da-proposta)

---

## 1. Fluxo da Jornada do Usuário

> **Tipo:** Fluxograma de negócio  
> **Propósito:** Jornada completa do cliente do início ao resultado.  
> **Decisão de design:** Validação de elegibilidade acontece antes de qualquer criação de recurso — se o cliente não for elegível, nenhuma conta é criada desnecessariamente.

```mermaid
flowchart TD
    A([🧑 Cliente acessa o sistema]) --> B[Preenche dados pessoais\nrenda, investimentos, tempo de conta]
    B --> C[Visualiza ofertas disponíveis\nde acordo com seu perfil]
    C --> D[Seleciona a oferta desejada]
    D --> E[Sistema exibe benefícios\ndisponíveis para a oferta escolhida]
    E --> F[Cliente seleciona os benefícios\nUI impede cashback + pontos juntos]
    F --> G[Submete a proposta]

    G --> H{Elegível para\na oferta escolhida?}
    H -->|Não| I([❌ Proposta negada\nCliente informado do motivo])

    H -->|Sim| J{Benefícios selecionados\nsão válidos para a oferta?}
    J -->|Não| K[Benefícios inválidos\nsão descartados]
    K --> L

    J -->|Sim| L[💳 Cria conta cartão]
    L --> M[🎁 Ativa benefícios elegíveis]
    M --> N[📝 Registra histórico da operação]
    N --> O[📬 Notifica cliente com o resultado]
    O --> P([✅ Proposta aprovada\nCartão criado + benefícios ativados])
```

---

## 2. Arquitetura de Microsserviços

> **Tipo:** Diagrama de componentes  
> **Propósito:** Visão dos serviços, responsabilidades e comunicação entre eles.
>
> **Por que microsserviços?**  
> Cada domínio — elegibilidade, cartão e benefícios — tem regras de negócio distintas e podem evoluir em ritmos diferentes. Separar em serviços independentes facilita manutenção e permite que cada um escale conforme sua demanda.
>
> **Por que Kafka?**  
> Notificação e registro de histórico não precisam acontecer antes de responder ao cliente. Publicar um evento no Kafka libera a resposta imediatamente e esses serviços processam no próprio ritmo, sem atrasar a experiência do usuário.

```mermaid
graph TB
    subgraph CLIENT["🖥️ Cliente"]
        WEB[Web App / Mobile]
    end

    subgraph GATEWAY["🔀 API Gateway"]
        GW[Autenticação JWT\nRoteamento\nRate Limiting]
    end

    subgraph SERVICES["⚙️ Microsserviços"]
        PROPOSAL[Proposal Service\nOrquestra o fluxo principal\nChain of Responsibility]
        ELIG[Eligibility Service\nValida elegibilidade\nStrategy Pattern]
        CARD[Card Service\nCria conta cartão]
        BEN[Benefit Service\nAtiva benefícios]
        NOTIF[Notification Service\nE-mail e SMS]
        AUDIT[Audit Service\nRegistra histórico]
    end

    subgraph KAFKA["📨 Apache Kafka"]
        T1[tópico: proposta-aprovada]
        T2[tópico: proposta-negada]
    end

    subgraph DATA["🗄️ Bancos de Dados"]
        DB1[(DB Proposal)]
        DB2[(DB Eligibility)]
        DB3[(DB Card)]
        DB4[(DB Benefit)]
        DB5[(DB Audit)]
    end

    WEB -->|HTTPS| GW
    GW --> PROPOSAL

    PROPOSAL -->|sync| ELIG
    PROPOSAL -->|sync| CARD
    PROPOSAL -->|sync| BEN
    PROPOSAL -->|publica evento| T1
    PROPOSAL -->|publica evento| T2

    T1 -->|consome| NOTIF
    T1 -->|consome| AUDIT
    T2 -->|consome| NOTIF
    T2 -->|consome| AUDIT

    PROPOSAL --> DB1
    ELIG --> DB2
    CARD --> DB3
    BEN --> DB4
    AUDIT --> DB5
```

---

## 3. Diagrama de Sequência

> **Tipo:** Sequence Diagram  
> **Propósito:** Ordem das chamadas entre os serviços ao longo do tempo.  
> **Ponto de atenção:** Notificação e auditoria são **assíncronas** — o cliente recebe a resposta sem precisar esperar por elas.

```mermaid
sequenceDiagram
    actor Cliente
    participant GW as API Gateway
    participant PS as Proposal Service
    participant ES as Eligibility Service
    participant CS as Card Service
    participant BS as Benefit Service
    participant KF as Kafka
    participant NS as Notification Service
    participant AS as Audit Service

    Cliente->>GW: POST /propostas\n{dados, oferta_id, beneficios[]}
    GW->>GW: Valida JWT
    GW->>PS: Encaminha requisição

    PS->>ES: Verifica elegibilidade\n{renda, investimentos, tempo_conta, oferta_id}

    alt Não elegível
        ES-->>PS: {elegivel: false, motivo: "RENDA_INSUFICIENTE"}
        PS->>KF: Publica em proposta-negada
        KF-->>NS: Consome → envia e-mail de negação
        KF-->>AS: Consome → registra histórico
        PS-->>GW: {status: "NEGADA", motivo}
        GW-->>Cliente: Proposta negada + motivo

    else Elegível
        ES-->>PS: {elegivel: true, beneficios_permitidos[]}
        PS->>CS: Cria conta cartão {cliente_id, oferta_id}
        CS-->>PS: {card_id, numero_mascarado}

        PS->>BS: Ativa benefícios {card_id, beneficios[]}
        BS-->>PS: {beneficios_ativados[]}

        PS->>KF: Publica em proposta-aprovada\n{card_id, beneficios_ativados}
        KF-->>NS: Consome → envia e-mail de aprovação
        KF-->>AS: Consome → registra histórico

        PS-->>GW: {status: "APROVADA", card_id, beneficios_ativados}
        GW-->>Cliente: ✅ Proposta aprovada + detalhes
    end
```

---

## 4. Strategy Pattern — Elegibilidade

> **Padrão:** Strategy (GoF — Comportamental)  
> **Onde:** `Eligibility Service`  
> **Problema resolvido:** Cada oferta tem critérios diferentes. Sem esse padrão, o código viraria um `if/else` enorme difícil de manter. Com Strategy, cada oferta é uma classe isolada com sua própria regra.  
> **Vantagem prática:** Quando surgir uma Oferta D, basta criar uma nova classe — sem alterar nenhuma regra existente. Isso segue o princípio **Open/Closed do SOLID**: aberto para extensão, fechado para modificação.

```mermaid
classDiagram
    class EligibilityService {
        -strategies: List~OfferStrategy~
        +getEligibleOffers(cliente): List~String~
        +validateBenefits(offerId, benefits): List~String~
    }

    class OfferStrategy {
        <<interface>>
        +isEligible(cliente: ClienteData): boolean
        +getOfferId(): String
        +getAllowedBenefits(): List~String~
    }

    class OfferAStrategy {
        +isEligible(cliente): boolean
        +getOfferId(): String
        +getAllowedBenefits(): List~String~
        -- Regra: renda maior que 1.000
        -- Benefícios: Cashback, Pontos
    }

    class OfferBStrategy {
        +isEligible(cliente): boolean
        +getOfferId(): String
        +getAllowedBenefits(): List~String~
        -- Regra: renda maior que 15.000
        --         E investimentos maior que 5.000
        -- Benefícios: Cashback, Pontos, Sala VIP
    }

    class OfferCStrategy {
        +isEligible(cliente): boolean
        +getOfferId(): String
        +getAllowedBenefits(): List~String~
        -- Regra: renda maior que 50.000
        --         E conta ativa há mais de 2 anos
        -- Benefícios: todos
    }

    EligibilityService --> OfferStrategy
    OfferStrategy <|.. OfferAStrategy
    OfferStrategy <|.. OfferBStrategy
    OfferStrategy <|.. OfferCStrategy
```

**Como funciona na prática:**

```mermaid
flowchart LR
    IN([Dados do cliente\n+ oferta solicitada]) --> SVC[EligibilityService]

    SVC --> SA[OfferAStrategy\n.isEligible]
    SVC --> SB[OfferBStrategy\n.isEligible]
    SVC --> SC[OfferCStrategy\n.isEligible]

    SA -->|true ou false| AGG[Filtra elegíveis\ne retorna lista]
    SB -->|true ou false| AGG
    SC -->|true ou false| AGG

    AGG --> OUT([Ofertas disponíveis\npara o cliente])
```

---

## 5. Chain of Responsibility — Validação

> **Padrão:** Chain of Responsibility (GoF — Comportamental)  
> **Onde:** `Proposal Service` — antes de iniciar qualquer processamento  
> **Problema resolvido:** A proposta precisa passar por várias validações diferentes. Colocar tudo num único método mistura responsabilidades e dificulta manutenção. Com esse padrão, cada validação é independente e pode barrar o fluxo sem que as outras precisem saber.  
> **Vantagem prática:** Para adicionar uma nova validação, basta criar um novo handler e encadear — sem mexer nos existentes.

```mermaid
flowchart TD
    REQ([📥 Proposta recebida]) --> H1

    H1[🔗 SchemaValidator\nCampos obrigatórios presentes?\nFormatos corretos?]
    H1 -->|❌ Inválido| E1[400 - Dados inválidos]
    H1 -->|✅ OK| H2

    H2[🔗 AuthValidator\nJWT válido?\nUsuário autenticado?]
    H2 -->|❌ Inválido| E2[401 - Não autorizado]
    H2 -->|✅ OK| H3

    H3[🔗 OfferEligibilityValidator\nCliente tem renda e condições\npara a oferta escolhida?]
    H3 -->|❌ Inelegível| E3[422 - Oferta não disponível]
    H3 -->|✅ OK| H4

    H4[🔗 BenefitConflictValidator\nCashback e Pontos\nforam selecionados juntos?]
    H4 -->|❌ Conflito| E4[422 - Benefícios incompatíveis]
    H4 -->|✅ OK| H5

    H5[🔗 BenefitOfferValidator\nOs benefícios escolhidos\nestão disponíveis para a oferta?]
    H5 -->|❌ Inválido| E5[422 - Benefício não disponível\npara esta oferta]
    H5 -->|✅ OK| DONE([✅ Proposta válida\nSegue para processamento])
```

---

## 6. Modelo de Dados

> **Tipo:** Diagrama Entidade-Relacionamento  
> **Decisão:** Cada microsserviço tem seu próprio banco — evita acoplamento entre serviços e permite que cada um evolua sua estrutura de dados de forma independente.  
> **Nota sobre IDs:** Cada tabela usa `BIGSERIAL` como chave primária interna (joins e índices) e `UUID` como identificador externo exposto na API — veja [ADR-004](decisions/ADR-004-uuid-strategy.md)

```mermaid
erDiagram
    PROPOSAL {
        bigserial id PK
        uuid public_id UK
        uuid cliente_id
        string oferta_id
        string status
        string motivo_negacao
        timestamp criado_em
        timestamp atualizado_em
    }

    CARD_ACCOUNT {
        bigserial id PK
        uuid public_id UK
        bigint proposal_id FK
        uuid conta_corrente_id
        string numero_mascarado
        string status
        string oferta_id
        timestamp criado_em
    }

    BENEFIT {
        bigserial id PK
        string codigo
        string nome
        string ofertas_permitidas
    }

    ACTIVATED_BENEFIT {
        bigserial id PK
        bigint card_account_id FK
        bigint benefit_id FK
        string status
        timestamp ativado_em
    }

    AUDIT_LOG {
        bigserial id PK
        uuid proposal_public_id
        string evento
        string descricao
        timestamp ocorrido_em
    }

    PROPOSAL ||--o| CARD_ACCOUNT : gera
    CARD_ACCOUNT ||--o{ ACTIVATED_BENEFIT : possui
    BENEFIT ||--o{ ACTIVATED_BENEFIT : ativado como
    PROPOSAL ||--o{ AUDIT_LOG : gera eventos
```

---

## 7. Estados da Proposta

> **Tipo:** Diagrama de estados  
> **Propósito:** Ciclo de vida de uma proposta do recebimento ao resultado final.

```mermaid
stateDiagram-v2
    [*] --> RECEBIDA : Cliente submete proposta

    RECEBIDA --> VALIDANDO : Inicia validações

    VALIDANDO --> NEGADA : Dados inválidos ou\ncritérios não atendidos
    VALIDANDO --> PROCESSANDO : Todas as validações passaram

    PROCESSANDO --> APROVADA : Cartão criado e\nbeneícios ativados
    PROCESSANDO --> ERRO : Falha técnica\nno processamento

    NEGADA --> NOTIFICADA : Cliente informado
    APROVADA --> NOTIFICADA : Cliente informado
    ERRO --> NOTIFICADA : Cliente informado

    NOTIFICADA --> [*]
```

---

*Os diagramas são renderizados automaticamente pelo GitHub.*  
*Para visualização isolada, use o [Mermaid Live Editor](https://mermaid.live).*
