# 08 — Google Play: publicação e política de permissões sensíveis

## Por que este app é sensível
`BIND_NOTIFICATION_LISTENER_SERVICE` é permissão restrita. O Google exige
justificativa funcional **essencial**. Rastreador de gastos por notificação é
um caso de uso **aceito** (a função principal *é* ler notificações), mas
precisa ser declarada corretamente.

## Pré-requisitos antes de submeter

### 1. Política de Privacidade pública (URL)
Hospede um markdown estático em GitHub Pages (grátis) com:
- O que o app coleta: notificações de pacotes selecionados.
- Onde fica: **exclusivamente no dispositivo do usuário**.
- O que NÃO faz: nenhuma transmissão para servidor, sem analytics, sem login.
- Como o usuário revoga: Configurações do Android.
- Contato para dúvidas.

Exemplo de texto pronto (adapte):
> O app Gastos processa localmente as notificações de bancos selecionados.
> Nenhum dado é transmitido a servidores. O usuário pode revogar o acesso
> a qualquer momento nas Configurações do Android. Não coletamos e-mail,
> nome ou identificadores.

### 2. Permissions Declaration Form
No Play Console → App content → Sensitive permissions → Notification access:
- Marque "core functionality".
- Descreva: "O app é um rastreador pessoal de gastos. A leitura de
  notificações bancárias é a funcionalidade principal; sem ela, o app não
  funciona. Nenhum dado sai do dispositivo."
- Anexar **vídeo demonstrativo** (30-60s) mostrando: permissão sendo
  concedida → notificação de banco chegando → transação aparecendo no app.
  Link do YouTube (não listado) é aceito.

### 3. Data Safety Form
- "Does your app collect or share any of the required user data types?" → **No**
  (dados não saem do dispositivo; o formulário se refere a coleta *para fora*).
- Se perguntarem sobre "app activity" ou "financial info": marque que
  processa localmente, sem transmissão. Quando em dúvida, seja conservador
  e explique no texto livre.

### 4. Conta de desenvolvedor
- Conta Google Play: R$ 25 (taxa única).
- Verificação de identidade obrigatória (documento + endereço).
- Para apps novos de devs pessoais: teste fechado com **mínimo 12 testers
  por 14 dias** antes de abrir para produção. **Planeje isso.**

## Roteiro de publicação

1. Criar app no Play Console.
2. Preencher Store Listing (título, descrição, screenshots 1080x1920, ícone 512x512).
3. Preencher **App content** completo (Privacy Policy, Data Safety, Sensitive Permissions, Content Rating).
4. Upload do AAB em **Internal testing** → testar você mesmo.
5. Promover para **Closed testing** → 12+ testers por 14 dias.
6. Promover para **Production** após aprovação.

## Riscos de rejeição
- **Falta de clareza** no vídeo: mostre o valor tangível (gasto aparecendo sozinho).
- **Descrição genérica** no Permissions Declaration: personalize.
- **Privacy Policy fraca**: deixe explícito que não há rede, sem analytics.
- **Não cumprir 12 testers/14 dias**: planeje o cronograma.

## Checklist de submissão
- [ ] Política de Privacidade publicada e no Store Listing
- [ ] Permissions Declaration preenchida com vídeo
- [ ] Data Safety preenchida
- [ ] Screenshots nas densidades exigidas
- [ ] AAB assinado gerado
- [ ] 12+ testers recrutados para closed testing
- [ ] Cronograma: 14 dias de closed testing antes de Production
