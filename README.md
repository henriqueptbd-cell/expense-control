# Gastos — Documentação do MVP

Documentação estruturada para o MVP de um app Android nativo que registra
gastos a partir de notificações bancárias. Tudo local, sem backend, sem login.

## Índice

| # | Documento | Quando consultar |
|---|---|---|
| 00 | [Overview e Arquitetura](00_OVERVIEW_ARQUITETURA.md) | **Leia primeiro** — visão geral, decisões, estrutura |
| 01 | [Setup do Projeto](01_SETUP_PROJETO.md) | Fase 0 — criar o projeto Android |
| 02 | [Modelo de Dados (Room)](02_MODELO_DADOS_ROOM.md) | Fase 1 — tabelas e DAOs |
| 03 | [Parsers](03_PARSERS.md) | Fase 2 / 5 — regex por banco |
| 04 | [NotificationListenerService](04_NOTIFICATION_LISTENER.md) | Fase 3 — captura de notificações |
| 05 | [UI com Compose](05_UI_COMPOSE.md) | Fase 4 — telas |
| 06 | [Testes Locais](06_TESTES_LOCAIS.md) | Fases 2 / 5 — JVM sem emulador |
| 07 | [Build e Assinatura](07_BUILD_ASSINATURA.md) | Fase 6 — APK/AAB |
| 08 | [Política do Google Play](08_PLAY_STORE_POLITICA.md) | Fase 8 — submissão |
| 09 | [Roadmap do MVP](09_ROADMAP_MVP.md) | **Roteiro principal** |
| 10 | [Monetização](10_MONETIZACAO.md) | Pós-validação |

## Ordem de leitura obrigatória para agentes
`00 → 09`. Os demais são consulta por fase.

## Regras invioláveis
1. Pacote `parser/` **não importa** `android.*` — Kotlin puro, testável no JVM.
2. Nenhum parser novo vai para `service/` sem `./gradlew test` verde.
3. Nenhuma feature fora do escopo do MVP (ver doc 00).
4. Nada de backend, login, analytics ou anúncios.
5. Dados nunca saem do dispositivo.

## Como usar com agentes
Cada agente recebe **um doc** + estado atual do repo.
Só avança se o gate de aceitação (`09_ROADMAP_MVP.md`) da fase estiver verde.

Sequência sugerida de paralelização:

```
Agente A (bootstrap + data):  00 → 01 → 02
Agente B (parsers):           depende de 02 → 03 → 06
Agente C (service + UI):      depende de 02 → 04 → 05
Agente D (build + play):      07 → 08 (independente, pode começar cedo)
```
