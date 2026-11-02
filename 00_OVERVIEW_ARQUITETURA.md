# 00 — Overview e Arquitetura

## Objetivo do produto
App Android nativo que captura notificações de bancos/carteiras digitais via
`NotificationListenerService` e registra transações automaticamente. Tudo local
(SQLite/Room), sem backend, sem login.

## Escopo do MVP
- Permissão de acesso a notificações
- `NotificationListenerService` filtrando 2–3 pacotes
- Parser por banco baseado em regex
- Fallback "Outros" com texto bruto
- Lista de transações + edição manual

## Fora do escopo (MVP)
Gráficos, metas, exportação, nuvem, login, múltiplas contas, Open Finance, backend.

## Decisões de arquitetura

| Decisão | Escolha | Motivo |
|---|---|---|
| Linguagem | Kotlin | Padrão atual Android |
| UI | Jetpack Compose | Menos boilerplate que XML |
| Acesso a dados | DAO direto no ViewModel | MVP sem backend; Repository seria boilerplate inútil |
| Parser | Kotlin puro (JVM-safe) | Permite teste local via JUnit sem emulador |
| Escopo de módulos | Single module | Um dev, MVP rápido |
| Atividade | Single Activity + Compose Nav | Padrão Compose |
| DI | Manual (sem Hilt) | MVP não justifica Hilt |

## Camadas (conceituais, dentro de um único módulo `app`)

```
┌───────────────────────────────────────┐
│ UI (Compose + ViewModel)              │
├───────────────────────────────────────┤
│ Service (Android, orquestrador)       │
├───────────────────────────────────────┤
│ Parsers (Kotlin puro — testável JVM)  │  ← CORAÇÃO
├───────────────────────────────────────┤
│ Data (Room)                           │
└───────────────────────────────────────┘
```

**Regra de ouro:** o pacote `parser` NÃO importa nada de `android.*`.
Isso é o que permite testar local com `./gradlew test` sem emulador.

## Estrutura de pacotes

```
com.seuapp.gastos
├── data
│   ├── AppDatabase.kt
│   ├── TransactionDao.kt
│   ├── TransactionEntity.kt
│   └── TransactionType.kt
├── parser
│   ├── NotificationParser.kt       (interface — Kotlin puro)
│   ├── ParserRegistry.kt           (Kotlin puro)
│   ├── ParseResult.kt              (Kotlin puro)
│   ├── banks/
│   │   ├── NubankParser.kt
│   │   ├── InterParser.kt
│   │   ├── PicPayParser.kt
│   │   └── MercadoPagoParser.kt
│   └── util/
│       └── MoneyParser.kt          (Kotlin puro)
├── service
│   ├── TransactionNotificationListener.kt
│   └── NotificationAccessHelper.kt
├── ui
│   ├── MainActivity.kt
│   ├── MainViewModel.kt
│   ├── screens/
│   │   ├── PermissionScreen.kt
│   │   ├── TransactionListScreen.kt
│   │   └── EditTransactionDialog.kt
│   └── theme/
│       └── Theme.kt
└── GastosApp.kt                    (Application class)
```

## Fluxo de dados (runtime)

1. Usuário habilita acesso a notificações nas Configurações do Android.
2. Banco X dispara notificação.
3. `TransactionNotificationListener.onNotificationPosted(sbn)` é chamado.
4. Filtro: pacote em `monitoredPackages`? Se não, retorna.
5. Deduplicação: `dao.existsByNotificationKey(sbn.key)`? Se sim, retorna.
6. `ParserRegistry.findParser(pkg)?.parse(...)` → `TransactionEntity?`
7. Se `null` → fallback "Outros" com texto bruto.
8. `dao.insert(entity)` → Room emite novo `Flow` → UI atualiza.

## Fluxo de teste local (JVM, sem Android)

1. Dev escreve `NubankParserTest.kt` em `app/src/test/`.
2. Roda `./gradlew test`.
3. Testa entradas como `"Compra de R$ 25,90 aprovada no Nubank"`.
4. Verifica `amount`, `type`, `category`.
5. Ajusta regex até passar. **Sem emulador, sem device.**
