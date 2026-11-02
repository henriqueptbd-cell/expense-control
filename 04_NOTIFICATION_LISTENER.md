# 04 — NotificationListenerService

## Manifest (trecho)

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<application ...>
    <service
        android:name=".service.TransactionNotificationListener"
        android:exported="false"
        android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
        <intent-filter>
            <action android:name="android.service.notification.NotificationListenerService" />
        </intent-filter>
    </service>
</application>
```

**Importante:** `BIND_NOTIFICATION_LISTENER_SERVICE` NÃO é runtime permission.
O usuário habilita manualmente em `Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS`.

## `NotificationAccessHelper.kt`

```kotlin
package com.seuapp.gastos.service

import android.content.ComponentName
import android.content.Context
import android.content.Intent
import android.provider.Settings

object NotificationAccessHelper {

    fun isEnabled(context: Context): Boolean {
        val flat = Settings.Secure.getString(
            context.contentResolver,
            "enabled_notification_listeners"
        ) ?: return false
        val cn = ComponentName(context, TransactionNotificationListener::class.java)
        return flat.split(":").any { ComponentName.unflattenFromString(it) == cn }
    }

    fun openSettings(context: Context) {
        val intent = Intent(Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS)
            .addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        context.startActivity(intent)
    }
}
```

## `TransactionNotificationListener.kt`

```kotlin
package com.seuapp.gastos.service

import android.app.Notification
import android.service.notification.NotificationListenerService
import android.service.notification.StatusBarNotification
import android.util.Log
import com.seuapp.gastos.data.AppDatabase
import com.seuapp.gastos.data.TransactionEntity
import com.seuapp.gastos.data.TransactionType
import com.seuapp.gastos.parser.ParserRegistry
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.cancel
import kotlinx.coroutines.launch

class TransactionNotificationListener : NotificationListenerService() {

    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())

    private val monitoredPackages = setOf(
        "com.nubank.app",
        "com.bancointer.app",
        "br.com.intermedium",
        "com.picpay",
        "com.mercadopago.wallet"
    )

    override fun onNotificationPosted(sbn: StatusBarNotification?, rankingMap: RankingMap?) {
        sbn ?: return
        if (sbn.packageName !in monitoredPackages) return
        if (sbn.isOngoing) return
        if (sbn.notification.flags and Notification.FLAG_GROUP_SUMMARY != 0) return

        val extras = sbn.notification.extras ?: return
        val title = extras.getCharSequence(Notification.EXTRA_TITLE)?.toString().orEmpty()
        val text = extras.getCharSequence(Notification.EXTRA_TEXT)?.toString().orEmpty()
        if (text.isBlank() && title.isBlank()) return

        val pkg = sbn.packageName
        val key = sbn.key
        val postTime = sbn.postTime

        scope.launch {
            try {
                val dao = AppDatabase.getInstance(applicationContext).transactionDao()
                if (dao.existsByNotificationKey(key)) return@launch

                val parser = ParserRegistry.findParser(pkg)
                val parsed = parser?.parse(title, text)

                val entity = parsed?.let {
                    TransactionEntity(
                        timestamp = postTime,
                        amount = it.amount,
                        type = it.type,
                        bankPackage = pkg,
                        category = it.category,
                        rawText = it.rawText,
                        notificationKey = key
                    )
                } ?: TransactionEntity(
                    timestamp = postTime,
                    amount = 0.0,
                    type = TransactionType.UNKNOWN,
                    bankPackage = pkg,
                    category = "Outros",
                    rawText = "$title | $text",
                    notificationKey = key
                )

                dao.insert(entity)
                Log.d(TAG, "Inserido: $entity")
            } catch (e: Exception) {
                Log.e(TAG, "Falha ao processar notificação $key", e)
            }
        }
    }

    override fun onDestroy() {
        scope.cancel()
        super.onDestroy()
    }

    companion object {
        private const val TAG = "NLService"
    }
}
```

## Filtros aplicados (por quê)
- `packageName !in monitoredPackages` — evita processar WhatsApp, email etc.
- `isOngoing` — descarta notificações persistentes (foreground services).
- `FLAG_GROUP_SUMMARY` — descarta agrupamentos; o filho já vem separado.
- `text.isBlank() && title.isBlank()` — ignora notificações sem conteúdo.
- `existsByNotificationKey` — idempotência (serviço pode reconectar).

## Testando no device sem app de banco
Usar `adb` para simular notificações de pacotes arbitrários **não é possível**
diretamente (o sistema valida o pacote). Alternativas:
1. Instalar um app simples que emita notificação com o mesmo `packageName` (não viável em release).
2. Instalar app de teste com `packageName` diferente e adicionar esse pacote
   temporariamente em `monitoredPackages` só no build `debug`.
3. **Recomendado:** usar o próprio app do banco (logar com conta pessoal) e
   capturar notificações reais. Depois salvar o texto exato em testes JVM.

## Testando parsers localmente (sem emulador)
Ver `06_TESTES_LOCAIS.md`.
