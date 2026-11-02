# 09 — Roadmap do MVP (roteiro para agentes)

## Princípios do roteiro
- Cada fase entrega algo **verificável** (passa teste, roda no device, aparece na tela).
- Parsers nascem testados no JVM. Nunca conectar parser novo ao serviço sem teste.
- Nada de features fora do escopo do MVP.
- Fases são **independentes o suficiente** para agentes paralelos em partes.

## Fase 0 — Bootstrap (1 dia)
Entregável: projeto compila e roda "Hello World".
- [ ] Criar projeto conforme `01_SETUP_PROJETO.md`
- [ ] `./gradlew assembleDebug` verde
- [ ] Rodar no emulador
- [ ] Commit inicial

## Fase 1 — Modelo de dados (meio dia)
Entregável: Room configurado, teste instrumentado do DAO (opcional nesta fase).
- [ ] Implementar `TransactionType`, `TransactionEntity`, `TransactionDao`, `AppDatabase`, `Categories`
- [ ] `./gradlew assembleDebug` verde
- [ ] Teste manual: inserir transação dummy via Logcat e ver tabela criada

## Fase 2 — Parsers JVM (2–3 dias)
Entregável: parser do Nubank + Inter passando em JUnit.
- [ ] `ParseResult`, `NotificationParser`, `MoneyParser`, `ParserRegistry`
- [ ] `NubankParser` + `NubankParserTest` (mín. 4 casos)
- [ ] `InterParser` + `InterParserTest` (mín. 3 casos)
- [ ] `./gradlew test` verde
- [ ] **Só depois disso** siga para Fase 3

## Fase 3 — NotificationListenerService (1–2 dias)
Entregável: notificação real de um banco vira linha no banco de dados (verificar via App Inspection).
- [ ] `TransactionNotificationListener`, `NotificationAccessHelper`
- [ ] Manifest com `<service>` correto
- [ ] Habilitar no device, disparar notificação real de teste, verificar insert
- [ ] Testar deduplicação (não insere duas vezes)

## Fase 4 — UI mínima (2 dias)
Entregável: abrir o app, ver as transações capturadas, editar uma.
- [ ] `MainActivity`, `MainViewModel`
- [ ] `PermissionScreen`, `TransactionListScreen`, `EditTransactionDialog`
- [ ] Voltar das Settings → lista aparece
- [ ] Edição persiste

## Fase 5 — Expansão de cobertura (3–5 dias)
Entregável: 3–4 bancos cobertos, cada um com testes verdes.
- [ ] Coletar notificações reais de cada banco (screenshots/prints do usuário)
- [ ] Adicionar `PicPayParser`, `MercadoPagoParser` (ou outros)
- [ ] Testes verdes para cada um
- [ ] Cobertura de casos de borda: valor sem centavos, notificação agrupada, spam

## Fase 6 — Polimento (2–3 dias)
Entregável: APK release pronto para amigos.
- [ ] Keystore gerado, configurado
- [ ] `isMinifyEnabled = true` e testar com R8 ativo
- [ ] Ícone do app, nome definitivo
- [ ] Tela de boas-vindas / onboarding curto (2 telas)
- [ ] Build `assembleRelease` testado em device físico

## Fase 7 — Alpha com amigos (2 semanas)
Entregável: 5 pessoas usando, feedback coletado.
- [ ] Distribuir APK via Firebase App Distribution ou Google Drive
- [ ] Coletar feedback estruturado (form simples)
- [ ] Corrigir bugs de parsing reportados
- [ ] **Não** adicionar features novas durante esta fase

## Fase 8 — Preparação Play Store (2 semanas, sobreposta à fase 7)
Entregável: submissão pronta.
- [ ] Política de Privacidade publicada (GitHub Pages)
- [ ] Permissions Declaration + vídeo
- [ ] Data Safety
- [ ] Screenshots, ícone, descrição
- [ ] AAB release assinado
- [ ] Closed testing: 12 testers por 14 dias

## Fase 9 — Lançamento público
- [ ] Promover para Production
- [ ] Acompanhar reviews nas primeiras 48h
- [ ] Corrigir crash loops imediatos

## Estimativa total (dev solo, pouco tempo/dia)
- Fases 0–4 (MVP funcional): **~1,5–2 semanas** de trabalho efetivo
- Fases 5–6 (cobertura + build): **~1 semana**
- Fase 7–8 (alpha + Play Store): **~2–4 semanas calendar**
- **Primeiro MVP instalável: 2 semanas. Primeiro público na Play Store: ~1,5–2 meses.**

## Dependências entre fases

```
0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9
                ↑
              (UI só depois do serviço
               funcionar por baixo)
```

Fases 5 e 8 são **paralelizáveis** com outras (agentes podem trabalhar em
novos parsers enquanto você cuida do Play Console).

## Regras de aceitação por fase (gate)
- Fase 2 e 5: **todos os testes JVM verdes** antes de avançar.
- Fase 3: transação real inserida no Room via App Inspector.
- Fase 4: edição de categoria persiste após matar o app.
- Fase 6: APK release instalado em device físico com R8 ativo.
