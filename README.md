# 💳 Solicitação de Cartão de Crédito — Arquitetura de Solução

> Teste Técnico — Arquitetura de Solução

---

## 📌 Visão Geral

Arquitetura para o fluxo de solicitação de cartão de crédito, onde o cliente escolhe benefícios durante a jornada respeitando critérios de elegibilidade por oferta.

A solução foi desenhada com **microsserviços** para separar responsabilidades de forma clara, e **Kafka** para comunicação assíncrona nos passos que não precisam bloquear a resposta ao cliente — como notificação e registro de histórico.

---

## 🗂️ Estrutura do Repositório

```
credit-card-architecture/
├── README.md
└── docs/
    ├── architecture.md              # Todos os diagramas
    ├── business-rules.md            # Regras de negócio detalhadas
    └── decisions/
        ├── ADR-001-microservices.md
        ├── ADR-002-kafka.md
        └── ADR-003-strategy-pattern.md
        └── ADR-004-uuid-strategy.md

```

---

## 🎯 Ofertas de Cartão

| Oferta | Critérios |
|--------|-----------|
| **Oferta A** | Renda > R$ 1.000 |
| **Oferta B** | Renda > R$ 15.000 **e** Investimentos > R$ 5.000 |
| **Oferta C** | Renda > R$ 50.000 **e** Conta corrente ativa há > 2 anos |

---

## 🎁 Benefícios

| Benefício | Regra |
|-----------|-------|
| **Cashback** | Todas as ofertas — incompatível com Pontos |
| **Pontos** | Todas as ofertas — incompatível com Cashback |
| **Sala VIP** | Apenas Ofertas B e C |
| **Seguro Viagem** | Apenas Oferta C |

---

## 🏛️ Principais Decisões

| Decisão | Motivo |
|---------|--------|
| Microsserviços | Cada domínio (elegibilidade, cartão, benefícios) tem responsabilidade própria e pode evoluir de forma independente |
| Kafka | Notificação e histórico não precisam bloquear a resposta ao cliente — processados de forma assíncrona |
| Strategy Pattern | Regras de elegibilidade isoladas por oferta — fácil de manter e de adicionar novas ofertas |
| Chain of Responsibility | Validações encadeadas e independentes — cada uma pode barrar o fluxo sem afetar as outras |

---

## 🔐 Dados Sensíveis

- Toda comunicação via HTTPS
- CPF e dados financeiros não aparecem em logs
- Autenticação via JWT
- Número do cartão armazenado criptografado

---

## 🛠️ Stack Sugerida

| Camada | Tecnologia |
|--------|-----------|
| API Gateway | Kong ou AWS API Gateway |
| Microsserviços | Spring Boot ou Node.js |
| Mensageria | Apache Kafka |
| Banco de dados | PostgreSQL |
| Autenticação | JWT |
