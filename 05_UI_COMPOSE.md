# 05 — UI com Jetpack Compose

## Telas (single-activity, 2 screens + 1 dialog)
1. **PermissionScreen** — se `NotificationAccessHelper.isEnabled() == false`.
2. **TransactionListScreen** — lista de transações.
3. **EditTransactionDialog** — edição de categoria/valor.

## `MainActivity.kt`

```kotlin
package com.seuapp.gastos.ui

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.lifecycle.viewmodel.compose.viewModel
import com.seuapp.gastos.service.NotificationAccessHelper
import com.seuapp.gastos.ui.screens.PermissionScreen
import com.seuapp.gastos.ui.screens.TransactionListScreen
import com.seuapp.gastos.ui.theme.GastosTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            GastosTheme {
                val vm: MainViewModel = viewModel()
                val hasAccess = NotificationAccessHelper.isEnabled(this)
                if (!hasAccess) PermissionScreen() else TransactionListScreen(vm)
            }
        }
    }

    override fun onResume() {
        super.onResume()
        // Recompõe quando o usuário volta das Configurações.
        // Em produção, usar um Flow sobre o Secure setting ou Lifecycle observer.
        recreate()
    }
}
```

## `MainViewModel.kt`

```kotlin
package com.seuapp.gastos.ui

import android.app.Application
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.viewModelScope
import com.seuapp.gastos.data.AppDatabase
import com.seuapp.gastos.data.TransactionEntity
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

class MainViewModel(app: Application) : AndroidViewModel(app) {
    private val dao = AppDatabase.getInstance(app).transactionDao()

    val transactions: StateFlow<List<TransactionEntity>> =
        dao.getAll().stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

    fun update(t: TransactionEntity) = viewModelScope.launch { dao.update(t) }
    fun delete(id: Long) = viewModelScope.launch { dao.deleteById(id) }
}
```

## `screens/PermissionScreen.kt`

```kotlin
package com.seuapp.gastos.ui.screens

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import com.seuapp.gastos.service.NotificationAccessHelper

@Composable
fun PermissionScreen() {
    val ctx = LocalContext.current
    Column(
        modifier = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Acesso às notificações", style = MaterialTheme.typography.headlineSmall)
        Spacer(Modifier.height(12.dp))
        Text("Para registrar gastos automaticamente, o app precisa ler notificações dos seus bancos. Tudo fica no seu celular.")
        Spacer(Modifier.height(24.dp))
        Button(onClick = { NotificationAccessHelper.openSettings(ctx) }) {
            Text("Abrir configurações")
        }
    }
}
```

## `screens/TransactionListScreen.kt`

```kotlin
package com.seuapp.gastos.ui.screens

import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import com.seuapp.gastos.data.TransactionEntity
import com.seuapp.gastos.data.TransactionType
import com.seuapp.gastos.ui.MainViewModel
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TransactionListScreen(vm: MainViewModel) {
    val list by vm.transactions.collectAsState()
    var editing by remember { mutableStateOf<TransactionEntity?>(null) }

    Scaffold(topBar = { TopAppBar(title = { Text("Gastos") }) }) { padding ->
        if (list.isEmpty()) {
            Box(Modifier.fillMaxSize().padding(padding), contentAlignment = Alignment.Center) {
                Text("Nenhuma transação capturada ainda.")
            }
        } else {
            LazyColumn(Modifier.fillMaxSize().padding(padding)) {
                items(list, key = { it.id }) { t ->
                    TransactionRow(t, onClick = { editing = t })
                    HorizontalDivider()
                }
            }
        }
    }

    editing?.let { t ->
        EditTransactionDialog(
            initial = t,
            onDismiss = { editing = null },
            onSave = { vm.update(it); editing = null }
        )
    }
}

@Composable
private fun TransactionRow(t: TransactionEntity, onClick: () -> Unit) {
    val fmt = remember { SimpleDateFormat("dd/MM HH:mm", Locale.getDefault()) }
    ListItem(
        headlineContent = { Text(t.category, fontWeight = FontWeight.SemiBold) },
        supportingContent = { Text("${fmt.format(Date(t.timestamp))} • ${t.bankPackage}") },
        trailingContent = {
            val sign = if (t.type == TransactionType.CREDIT) "+" else "-"
            Text("$sign R$ %.2f".format(t.amount))
        },
        modifier = Modifier.clickable(onClick = onClick)
    )
}
```

## `screens/EditTransactionDialog.kt`

```kotlin
package com.seuapp.gastos.ui.screens

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.seuapp.gastos.data.Categories
import com.seuapp.gastos.data.TransactionEntity

@Composable
fun EditTransactionDialog(
    initial: TransactionEntity,
    onDismiss: () -> Unit,
    onSave: (TransactionEntity) -> Unit
) {
    var category by remember { mutableStateOf(initial.category) }
    var amountText by remember { mutableStateOf(initial.amount.toString()) }
    var expanded by remember { mutableStateOf(false) }

    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("Editar transação") },
        text = {
            Column {
                OutlinedTextField(
                    value = amountText,
                    onValueChange = { amountText = it },
                    label = { Text("Valor") },
                    modifier = Modifier.fillMaxWidth()
                )
                Spacer(Modifier.height(12.dp))
                ExposedDropdownMenuBox(expanded = expanded, onExpandedChange = { expanded = it }) {
                    OutlinedTextField(
                        value = category,
                        onValueChange = {},
                        readOnly = true,
                        label = { Text("Categoria") },
                        trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded) },
                        modifier = Modifier.menuAnchor().fillMaxWidth()
                    )
                    ExposedDropdownMenu(expanded = expanded, onDismissRequest = { expanded = false }) {
                        Categories.ALL.forEach { c ->
                            DropdownMenuItem(text = { Text(c) }, onClick = {
                                category = c; expanded = false
                            })
                        }
                    }
                }
            }
        },
        confirmButton = {
            TextButton(onClick = {
                val newAmount = amountText.replace(",", ".").toDoubleOrNull() ?: initial.amount
                onSave(initial.copy(amount = newAmount, category = category, isManualOverride = true))
            }) { Text("Salvar") }
        },
        dismissButton = { TextButton(onClick = onDismiss) { Text("Cancelar") } }
    )
}
```

## Checklist de UI
- [ ] PermissionScreen aparece quando acesso está desligado
- [ ] Ao voltar das Configurações, lista aparece
- [ ] Lista atualiza em tempo real quando notificação chega
- [ ] Dialog edita categoria e valor
- [ ] `isManualOverride` marcado após edição
