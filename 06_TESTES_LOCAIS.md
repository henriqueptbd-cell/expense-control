# 06 — Testes Locais (JVM, sem emulador)

## Estratégia
A camada `parser` é Kotlin puro → roda em `test/` (JVM) com JUnit + Truth.
Não precisa de emulador, nem de device. Você roda `./gradlew test` e itera
em segundos.

## Estrutura

```
app/src/test/java/com/seuapp/gastos/
├── parser/
│   ├── MoneyParserTest.kt
│   ├── ParserRegistryTest.kt
│   └── banks/
│       ├── NubankParserTest.kt
│       ├── InterParserTest.kt
│       ├── PicPayParserTest.kt
│       └── MercadoPagoParserTest.kt
```

## `MoneyParserTest.kt`

```kotlin
package com.seuapp.gastos.parser

import com.google.common.truth.Truth.assertThat
import com.seuapp.gastos.parser.util.MoneyParser
import org.junit.Test

class MoneyParserTest {
    @Test fun `extrai valor simples`() {
        assertThat(MoneyParser.extract("Compra de R$ 25,90 aprovada")).isEqualTo(25.90)
    }
    @Test fun `extrai valor com milhar`() {
        assertThat(MoneyParser.extract("R$ 1.234,56")).isEqualTo(1234.56)
    }
    @Test fun `retorna null sem valor`() {
        assertThat(MoneyParser.extract("Você tem uma nova mensagem")).isNull()
    }
}
```

## `NubankParserTest.kt` (modelo)

```kotlin
package com.seuapp.gastos.parser.banks

import com.google.common.truth.Truth.assertThat
import com.seuapp.gastos.data.TransactionType
import org.junit.Test

class NubankParserTest {
    private val parser = NubankParser()

    @Test fun `reconhece pacote`() {
        assertThat(parser.canHandle("com.nubank.app")).isTrue()
        assertThat(parser.canHandle("com.outro")).isFalse()
    }

    @Test fun `compra é débito`() {
        val r = parser.parse("Nubank", "Compra de R$ 25,90 aprovada no crédito")
        assertThat(r).isNotNull()
        assertThat(r!!.amount).isEqualTo(25.90)
        assertThat(r.type).isEqualTo(TransactionType.DEBIT)
    }

    @Test fun `recebimento é crédito`() {
        val r = parser.parse("Nubank", "Você recebeu R$ 100,00 via Pix")
        assertThat(r).isNotNull()
        assertThat(r!!.amount).isEqualTo(100.0)
        assertThat(r.type).isEqualTo(TransactionType.CREDIT)
    }

    @Test fun `sem valor retorna null`() {
        assertThat(parser.parse("Nubank", "Sua fatura fechou")).isNull()
    }
}
```

## Regra de trabalho (obrigatória no roadmap)
1. **Antes** de tocar em `service/` ou `ui/`, o parser correspondente precisa
   ter testes verdes no JVM.
2. Cole no teste o **texto literal** de uma notificação real (screenshot/print).
3. Se regex falhar, ajuste `NubankParser.parse` até o teste passar.
4. Só então conecte no `NotificationListenerService` para validação manual no device.

## Comandos úteis

```bash
# rodar todos os testes JVM
./gradlew test

# rodar só um teste
./gradlew test --tests "com.seuapp.gastos.parser.banks.NubankParserTest"

# ver relatório HTML
open app/build/reports/tests/testDebugUnitTest/index.html
```

## Testes instrumentados (adicionar só na fase 4)
Testes instrumentados (`androidTest/`) exigem emulador e são lentos. **Adiar**
até o MVP estar em uso real; não bloqueiam nada agora.
