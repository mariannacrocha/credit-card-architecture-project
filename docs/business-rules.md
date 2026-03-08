# 📋 Regras de Negócio

---

## Ofertas de Cartão

### Oferta A
- **Critério:** Renda comprovada acima de R$ 1.000
- **Benefícios disponíveis:** Cashback, Pontos

### Oferta B
- **Critérios (ambos obrigatórios):**
  - Renda comprovada acima de R$ 15.000
  - Investimentos ativos acima de R$ 5.000
- **Benefícios disponíveis:** Cashback, Pontos, Sala VIP

### Oferta C
- **Critérios (ambos obrigatórios):**
  - Renda comprovada acima de R$ 50.000
  - Conta corrente ativa há mais de 2 anos
- **Benefícios disponíveis:** Cashback, Pontos, Sala VIP, Seguro Viagem

---

## Regras de Benefícios

| Benefício | Disponível para | Restrição |
|-----------|----------------|-----------|
| Cashback | Ofertas A, B e C | Incompatível com Pontos |
| Pontos | Ofertas A, B e C | Incompatível com Cashback |
| Sala VIP | Ofertas B e C | Nenhuma |
| Seguro Viagem | Oferta C | Nenhuma |

---

## Premissas

- A conta corrente do cliente já existe — o sistema cria apenas a conta cartão
- Um cliente pode ser elegível a mais de uma oferta — o sistema apresenta todas e o cliente escolhe
- A validação de benefícios ocorre no backend mesmo que a interface já impeça seleções inválidas
- Dados de renda e investimentos fornecidos pelo cliente no momento da solicitação
