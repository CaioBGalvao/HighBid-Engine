# HighBid Engine: Arquitetura e Planejamento Teórico

## Visão Geral

Sistema de leilões em tempo real onde qualquer usuário pode se registrar, criar leilões (com foto, título, descrição, preço inicial e horário de início) e participar de leilões ativos. O grande desafio deste projeto é manter a consistência de lances concorrentes em larga escala e atualizar os clientes em tempo real.

## 1. Atores e Tipos de Usuários

- **Visitante**: Pode navegar e visualizar leilões, mas não pode dar lances.
- **Usuário Autenticado (Comprador/Vendedor)**: Pode criar novos leilões e dar lances em leilões de terceiros. Pode "seguir" leilões.
- **Administrador**: Moderação de conteúdo, cancelamento de leilões fraudulentos e banimento de usuários.

## 2. Controle de Concorrência (O Coração do Motor de Lances)

Quando milhares de usuários tentam dar um lance ao mesmo tempo, precisamos garantir que:

1. Ninguém consiga dar um lance menor ou igual ao lance vencedor atual.
2. O banco de dados não sofra um "Deadlock" ou esgotamento de conexões (Connection Pool Exhaustion).

**Estratégia Proposta (Assíncrona e Otimizada para Performance):**

- **Fase 1 (Gatekeeper no Redis)**: Quando a request de lance chega, um script Lua no Redis verifica o `highest_bid` atual. Se o novo lance for menor, é rejeitado imediatamente na memória (latência < 1ms), protegendo o banco de dados.
- **Fase 2 (Processamento Assíncrono via Fila)**: Se aprovado pelo Redis, o fluxo é totalmente assíncrono. O backend retorna um `202 Accepted` para o usuário imediatamente. O lance entra em uma fila de alta prioridade (Laravel Horizon no Redis).
- **Fase 3 (Consolidação)**: Os workers da fila pegam o lance, salvam no banco de dados (PostgreSQL) garantindo consistência, e então disparam um evento no Reverb para atualizar todos os clientes conectados de que o lance foi confirmado.

## 2.1. Tipos de Lances e Incrementos

- **Incremento Padrão (Quick Bid)**: O leilão pode ter um incremento configurado (ex: + R$ 5,00). O usuário clica em um botão e o sistema envia o lance usando o valor atual + incremento.
- **Proxy Bidding (Lance com Valor Máximo)**: O usuário informa o valor MÁXIMO que está disposto a pagar. O sistema automaticamente dá lances em nome dele usando o menor incremento possível até o limite estipulado, sempre que alguém tentar superá-lo. Isso evita que o usuário precise acompanhar o leilão a todo momento.

## 3. Comunicação em Tempo Real (WebSocket vs SSE)

- **Laravel Reverb (WebSocket)**:
  - Usado na **sala de leilão ao vivo** (auction room).
  - Mantém conexão persistente ativa. Subscrição em canais privados/focados (`auction.{id}`).
  - Responsável por transmitir lances recebidos em tempo real, timer regressivo exato e mudança de estado ("Vendido!").
- **Mercure (Server-Sent Events - SSE via FrankenPHP)**:
  - Usado para **notificações globais e assíncronas** de usuários que não estão na sala de leilão. Estará integrado nativamente através do webserver Caddy do FrankenPHP.
  - Exemplo: Alerta de "Seu leilão seguido começa em 5 minutos", "Você foi superado em um leilão" (se o usuário estiver na home), "Leilão X foi arrematado".
  - Muito eficiente para transmitir alertas sem custo de manter um socket bidirecional stateful pesado para usuários inativos.

## 4. Proteção contra "Legitimate DDoS" (Thundering Herd Problem)

- Ocorre quando um leilão popular termina, ou quando todos dão lances no último segundo (Sniping).
- **Proteções**:
  - **Rate Limiting por Usuário/Leilão**: Um usuário não pode dar mais de X lances por segundo.
  - **Anti-Sniping (Soft Close)**: Para evitar que pessoas roubem o leilão no último segundo, aplicaremos a regra de Soft Close oficial: Se um lance for dado nos últimos X minutos (ex: 3 minutos), o relógio do leilão é automaticamente estendido para +X minutos, permitindo que outros usuários tenham tempo de reagir.
  - **Validação off-loaded**: Usar validação rigorosa no Form Request e rejeitar no nível do Proxy/Nginx/FrankenPHP requests malformados antes de carregar o framework todo.

## 5. Estrutura de Banco de Dados (Resumo)

- `users`: (id, name, email, password, etc)
- `auctions`: (id, user_id, title, description, start_price, current_price, starts_at, ends_at, status)
- `bids`: (id, auction_id, user_id, amount, is_proxy, max_proxy_amount, created_at) - Indexado por `(auction_id, amount desc)`.
- `followers`: (user_id, auction_id) para notificações.
- `media`: (Spatie Media Library para fotos dos itens).
