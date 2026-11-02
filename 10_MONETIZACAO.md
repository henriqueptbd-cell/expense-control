# 10 — Monetização (futuro, pós-validação)

## Estratégia
Freemium: versão gratuita suficiente para uso pessoal básico; versão paga
remove limite e desbloqueia conforto.

## Proposta inicial (a validar após 3 meses de uso)

| Recurso | Grátis | Pago |
|---|---|---|
| Bancos suportados | 2 (Nubank + Inter) | Todos |
| Transações/mês | 50 | Ilimitado |
| Edição manual | ✅ | ✅ |
| Categorias customizadas | ❌ | ✅ |
| Exportação CSV | ❌ | ✅ |
| Backup local | ❌ | ✅ |
| Backup criptografado | ❌ | ✅ |

**Sem backend.** Pago habilitado via Play Billing. Nenhuma chamada de rede
para servidor próprio. Play Billing valida o receipt localmente (cache).

## Modelo de preço (definido em conversa anterior)
- Assinatura: **R$ 9,90/mês**
- Compra única (lifetime): **R$ 19,90**

Recomendação: **começar só com compra única** no primeiro release pago. Menos
fricção, menos complexidade de Play Billing recorrente, menos cancelamento
para gerenciar. Migrar para assinatura só se houver demanda demonstrada.

## Play Billing — escopo técnico mínimo
- `com.android.billingclient:billing-ktx`
- Um `BillingClient` singleton, `queryPurchases`, `acknowledgePurchase`.
- Persistir flag `isPro: Boolean` em `SharedPreferences` ou nova tabela Room.
- Checagem **offline**: cache local da última verificação + refresh quando online.

## O que NÃO fazer
- ❌ Backend de licenciamento (contraria o "tudo local" que é argumento de venda)
- ❌ Login obrigatório para ativar Pro
- ❌ Analytics de uso (contraria privacidade)
- ❌ Coleta de email
- ❌ Anúncios (mata a proposta de valor)

## Quando ativar monetização
- Após **500+ instalações** com retenção D30 acima de 20%.
- Após receber pedidos espontâneos de recurso (ex: "queria exportar").
- **Nunca** antes de ter uso real estável por 2–3 meses.
