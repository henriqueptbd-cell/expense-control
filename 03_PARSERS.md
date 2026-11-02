# 03 — Parsers (Kotlin puro, testável no JVM)

## Princípio arquitetural
Parsers NÃO importam `android.*`. São Kotlin puro. Isso permite `./gradlew test`
no JVM, sem emulador. Este é o principal mecanismo para "testar local antes do Android".

## `ParseResult.kt`

```kotlin
package com.seuapp.gastos.parser

import com.seuapp.gastos.data.TransactionType

data class ParseResult(
    val amount: Double,
    val type: TransactionType,
    val category: String,
    val rawText: String
)
```

## `NotificationParser.kt` (interface)

```kotlin
package com.seuapp.gastos.parser

interface NotificationParser {
    /** Retorna true se este parser sabe processar notificações deste pacote. */
    fun canHandle(packageName: String): Boolean

    /** Retorna ParseResult se reconheceu, ou null para fallback. */
    fun parse(title: String, text: String): ParseResult?
}
```

## `MoneyParser.kt` (utilitário compartilhado)

```kotlin
package com.seuapp.gastos.parser.util

object MoneyParser {
    /**
     * Converte "R$ 1.234,56" ou "R$ 25,90" ou "R$ 5,00" em Double.
     * Aceita também "RS" e "R $" por tolerância.
     * Retorna null se não conseguir.
     */
    private val regex = Regex("""R\$\s*([\d.]+,\d{2})""", RegexOption.IGNORE_CASE)

    fun extract(text: String): Double? {
        val m = regex.find(text) ?: return null
        val raw = m.groupValues[1]
        val normalized = raw.replace(".", "").replace(",", ".")
        return normalized.toDoubleOrNull()
    }
}
```

## `ParserRegistry.kt`

```kotlin
package com.seuapp.gastos.parser

import com.seuapp.gastos.parser.banks.*

object ParserRegistry {
    private val parsers: List<NotificationParser> = listOf(
        NubankParser(),
        InterParser(),
        PicPayParser(),
        MercadoPagoParser()
    )

    fun findParser(packageName: String): NotificationParser? =
        parsers.firstOrNull { it.canHandle(packageName) }

    fun all(): List<NotificationParser> = parsers
}
```

## `banks/NubankParser.kt`

```kotlin
package com.seuapp.gastos.parser.banks

import com.seuapp.gastos.data.TransactionType
import com.seuapp.gastos.parser.NotificationParser
import com.seuapp.gastos.parser.ParseResult
import com.seuapp.gastos.parser.util.MoneyParser

class NubankParser : NotificationParser {

    override fun canHandle(packageName: String) = packageName == "com.nubank.app"

    override fun parse(title: String, text: String): ParseResult? {
        val full = "$title $text".trim()
        val amount = MoneyParser.extract(full) ?: return null

        val lower = full.lowercase()

        val type = when {
            listOf("compra", "pagamento", "pago", "enviad").any { lower.contains(it) } -> TransactionType.DEBIT
            listOf("recebeu", "recebido", "crédito", "entrada").any { lower.contains(it) } -> TransactionType.CREDIT
            else -> TransactionType.UNKNOWN
        }

        val category = when {
            listOf("ifood", "uber eats", "restaurante", "padaria", "mercado").any { lower.contains(it) } -> "Alimentação"
            listOf("uber", "99", "posto", "combustível").any { lower.contains(it) } -> "Transporte"
            listOf("farmácia", "drogaria", "hospital").any { lower.contains(it) } -> "Saúde"
            type == TransactionType.CREDIT -> "Transferência"
            else -> "Outros"
        }

        return ParseResult(amount, type, category, full)
    }
}
```

## `banks/InterParser.kt`

```kotlin
package com.seuapp.gastos.parser.banks

import com.seuapp.gastos.data.TransactionType
import com.seuapp.gastos.parser.NotificationParser
import com.seuapp.gastos.parser.ParseResult
import com.seuapp.gastos.parser.util.MoneyParser

class InterParser : NotificationParser {

    override fun canHandle(packageName: String) =
        packageName == "com.bancointer.app" || packageName == "br.com.intermedium"

    override fun parse(title: String, text: String): ParseResult? {
        val full = "$title $text".trim()
        val amount = MoneyParser.extract(full) ?: return null

        val lower = full.lowercase()
        val type = when {
            lower.contains("compra") || lower.contains("pix enviado") || lower.contains("pagamento") -> TransactionType.DEBIT
            lower.contains("receb") || lower.contains("pix recebido") -> TransactionType.CREDIT
            else -> TransactionType.UNKNOWN
        }

        return ParseResult(amount, type, "Outros", full)
    }
}
```

## `banks/PicPayParser.kt`

```kotlin
package com.seuapp.gastos.parser.banks

import com.seuapp.gastos.data.TransactionType
import com.seuapp.gastos.parser.NotificationParser
import com.seuapp.gastos.parser.ParseResult
import com.seuapp.gastos.parser.util.MoneyParser

class PicPayParser : NotificationParser {

    override fun canHandle(packageName: String) = packageName == "com.picpay"

    override fun parse(title: String, text: String): ParseResult? {
        val full = "$title $text".trim()
        val amount = MoneyParser.extract(full) ?: return null
        val lower = full.lowercase()
        val type = when {
            lower.contains("pagou") || lower.contains("enviou") || lower.contains("compra") -> TransactionType.DEBIT
            lower.contains("recebeu") || lower.contains("cashback") -> TransactionType.CREDIT
            else -> TransactionType.UNKNOWN
        }
        return ParseResult(amount, type, "Outros", full)
    }
}
```

## `banks/MercadoPagoParser.kt`

```kotlin
package com.seuapp.gastos.parser.banks

import com.seuapp.gastos.data.TransactionType
import com.seuapp.gastos.parser.NotificationParser
import com.seuapp.gastos.parser.ParseResult
import com.seuapp.gastos.parser.util.MoneyParser

class MercadoPagoParser : NotificationParser {

    override fun canHandle(packageName: String) = packageName == "com.mercadopago.wallet"

    override fun parse(title: String, text: String): ParseResult? {
        val full = "$title $text".trim()
        val amount = MoneyParser.extract(full) ?: return null
        val lower = full.lowercase()
        val type = when {
            lower.contains("pagamento") || lower.contains("compra") || lower.contains("envio") -> TransactionType.DEBIT
            lower.contains("receb") || lower.contains("entrada") -> TransactionType.CREDIT
            else -> TransactionType.UNKNOWN
        }
        return ParseResult(amount, type, "Outros", full)
    }
}
```

## Como adicionar um novo banco (checklist)
1. Criar `banks/NovoBancoParser.kt` implementando `NotificationParser`.
2. Adicionar em `ParserRegistry.parsers`.
3. Criar `NovoBancoParserTest.kt` com 3-5 casos (débito, crédito, sem valor).
4. Rodar `./gradlew test`. Só commitar após verde.

## Fonte de notificações reais para teste
- Peça ao usuário (você) prints/screenshots de notificações reais de cada banco.
- Cole o texto literal em `NubankParserTest.kt` como constante.
- Ajuste regex até passar. Isso é mais confiável do que adivinhar formatos.
