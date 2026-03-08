# ADR-002: Apache Kafka para Comunicação Assíncrona

**Status:** Aceito

---

## Contexto

Após aprovar uma proposta, o sistema precisa notificar o cliente e registrar o histórico. Essas ações não precisam acontecer antes de responder ao cliente — são "efeitos colaterais" da operação principal.

## Decisão

Usar Kafka para publicar eventos após o processamento principal. Notification Service e Audit Service consomem esses eventos de forma assíncrona.

## Motivo

- O cliente recebe a resposta mais rápido — não precisa esperar o e-mail ser enviado
- Se o serviço de notificação estiver fora do ar, a proposta não falha — o evento fica no Kafka e é processado quando o serviço voltar
- Fácil de adicionar novos consumidores no futuro (ex: analytics) sem alterar o fluxo principal

## Desvantagens reconhecidas

- A notificação pode chegar alguns segundos depois da resposta da API
- Kafka adiciona complexidade de infraestrutura
