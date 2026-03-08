# ADR-003: Strategy Pattern para Regras de Elegibilidade

**Status:** Aceito

---

## Contexto

Temos 3 ofertas com critérios de elegibilidade diferentes. A abordagem mais direta seria um `if/else` para cada oferta, mas isso concentra todas as regras num único lugar e torna difícil adicionar novas ofertas sem risco de quebrar as existentes.

## Decisão

Usar o padrão Strategy — cada oferta é uma classe separada que implementa uma interface comum com o método `isEligible`.

## Motivo

- Cada regra de oferta fica isolada e é fácil de testar unitariamente
- Para adicionar uma nova oferta, basta criar uma nova classe sem alterar as existentes
- Segue o princípio Open/Closed do SOLID

## Desvantagens reconhecidas

- Mais classes para gerenciar do que um `if/else` simples
- Para pouquíssimas regras estáticas, pode ser considerado over-engineering
