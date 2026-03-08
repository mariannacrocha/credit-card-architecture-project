# ADR-004: Estratégia de Identificadores — UUID + BIGSERIAL

**Status:** Aceito

---

## Contexto

Em bancos relacionais como PostgreSQL, a escolha do tipo de chave primária tem impacto direto na performance. Ao mesmo tempo, num contexto de microsserviços, precisamos de identificadores que possam ser gerados de forma independente por serviços diferentes, sem risco de colisão.

---

## O problema do UUID puro em bancos relacionais

O PostgreSQL mantém um índice ordenado na chave primária. Quando o ID é sequencial, cada novo registro é inserido no final do índice — previsível e rápido.

O UUID v4 é aleatório. Cada novo registro vai para uma posição diferente no índice, forçando o banco a reorganizar a estrutura a cada insert. Esse fenômeno se chama **index fragmentation** e causa impacto real de performance em tabelas com muitos registros.

Além disso, UUID ocupa **16 bytes** contra **8 bytes** do BIGINT — esse custo se multiplica em todas as foreign keys que referenciam a tabela.

```
ID sequencial:  1, 2, 3, 4, 5
→ sempre insere no final ✅ rápido e previsível

UUID v4:  a3f8..., 12bc..., f901..., 44de...
→ insere em posições aleatórias ❌ fragmenta o índice
```

---

## Decisão

Usar **dois identificadores** em cada tabela:

```sql
CREATE TABLE proposal (
    id        BIGSERIAL PRIMARY KEY,              -- chave interna, para joins e índices
    public_id UUID DEFAULT gen_random_uuid() UNIQUE, -- identificador externo, exposto na API
    ...
);
```

- `id BIGSERIAL` — chave primária interna, usada em joins e foreign keys dentro do banco. Sequencial, rápido, leve.
- `public_id UUID` — exposto na API e nos eventos do Kafka. Nunca expõe o `id` interno.

---

## Motivo

**Por que manter UUID externamente:**
- Em microsserviços, serviços diferentes podem gerar IDs sem precisar se coordenar
- ID sequencial expõe informações do negócio (ex: `id=1423` revela que existem 1422 propostas anteriores)
- UUID facilita idempotência — o cliente pode gerar o ID antes de chamar a API

**Por que usar BIGSERIAL internamente:**
- Índices em chaves sequenciais não fragmentam
- Ocupa menos espaço — impacta todas as foreign keys da tabela
- Joins internos são mais rápidos

---

## Alternativa considerada — UUID v7

O UUID v7 é ordenado por tempo, o que resolve o problema de fragmentação mantendo um único identificador:

```
UUID v4 (aleatório):  f47ac10b-58cc-4372-a567-0e02b2c3d479
UUID v7 (ordenado):   018e3b2a-f1c0-7000-8000-000000000001
                       ↑ prefixo com timestamp = ordenado no índice ✅
```

É uma alternativa válida e mais simples. Foi preterida aqui porque o suporte nativo no PostgreSQL ao UUID v7 ainda é recente — a abordagem BIGSERIAL + UUID é mais amplamente suportada e documentada.

---

## Desvantagens reconhecidas

- Duas colunas de ID por tabela aumentam um pouco a complexidade do schema
- É necessário garantir que o `id` interno nunca vaze para fora do serviço — apenas o `public_id` deve aparecer em respostas de API e eventos
