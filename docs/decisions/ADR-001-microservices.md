# ADR-001: Arquitetura em Microsserviços

**Status:** Aceito

---

## Contexto

O fluxo envolve domínios distintos: validação de elegibilidade, criação de cartão e ativação de benefícios. Cada um tem regras de negócio próprias e podem crescer de forma diferente ao longo do tempo.

## Decisão

Separar cada domínio em um microsserviço independente com seu próprio banco de dados.

## Motivo

- Cada serviço pode ser alterado e publicado sem afetar os outros
- Se o Benefit Service precisar escalar mais que os outros, isso é possível de forma independente
- Falha em um serviço não derruba toda a aplicação

## Desvantagens reconhecidas

- Mais complexidade operacional do que um monólito
- Comunicação entre serviços precisa ser bem gerenciada
