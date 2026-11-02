# 02 — Modelo de Dados (Room)

## Princípios
- Uma única tabela `transactions` no MVP. Categoria e banco como strings.
- Índice em `timestamp` para ordenação da lista.
- `notificationKey` único para deduplicação.
- Campos `rawText` e `isManualOverride` preparam o app para revisão e edição.

## `TransactionType.kt`

```kotlin
package com.seuapp.gastos.data

enum class TransactionType { DEBIT, CREDIT, UNKNOWN }
```

## `TransactionEntity.kt`

```kotlin
package com.seuapp.gastos.data

import androidx.room.Entity
import androidx.room.Index
import androidx.room.PrimaryKey

@Entity(
    tableName = "transactions",
    indices = [
        Index(value = ["timestamp"]),
        Index(value = ["notificationKey"], unique = true)
    ]
)
data class TransactionEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val timestamp: Long,
    val amount: Double,
    val type: TransactionType,
    val bankPackage: String,
    val category: String,
    val rawText: String? = null,
    val notificationKey: String,
    val isManualOverride: Boolean = false
)
```

## `TransactionDao.kt`

```kotlin
package com.seuapp.gastos.data

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import androidx.room.Update
import kotlinx.coroutines.flow.Flow

@Dao
interface TransactionDao {

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insert(transaction: TransactionEntity): Long

    @Update
    suspend fun update(transaction: TransactionEntity)

    @Query("SELECT * FROM transactions ORDER BY timestamp DESC")
    fun getAll(): Flow<List<TransactionEntity>>

    @Query("SELECT * FROM transactions WHERE id = :id LIMIT 1")
    suspend fun getById(id: Long): TransactionEntity?

    @Query("SELECT EXISTS(SELECT 1 FROM transactions WHERE notificationKey = :key)")
    suspend fun existsByNotificationKey(key: String): Boolean

    @Query("DELETE FROM transactions WHERE id = :id")
    suspend fun deleteById(id: Long)
}
```

## `AppDatabase.kt`

```kotlin
package com.seuapp.gastos.data

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(
    entities = [TransactionEntity::class],
    version = 1,
    exportSchema = true
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun transactionDao(): TransactionDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase =
            INSTANCE ?: synchronized(this) {
                INSTANCE ?: Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "gastos.db"
                ).build().also { INSTANCE = it }
            }
    }
}
```

## Categorias sugeridas (string constants)
```kotlin
object Categories {
    const val OUTROS = "Outros"
    const val ALIMENTACAO = "Alimentação"
    const val TRANSPORTE = "Transporte"
    const val SAUDE = "Saúde"
    const val LAZER = "Lazer"
    const val COMPRAS = "Compras"
    const val MORADIA = "Moradia"
    const val SALARIO = "Salário"
    const val TRANSFERENCIA = "Transferência"
    val ALL = listOf(OUTROS, ALIMENTACAO, TRANSPORTE, SAUDE, LAZER, COMPRAS, MORADIA, SALARIO, TRANSFERENCIA)
}
```

## Nota sobre migrações
Na v1 não há migração. A partir da v2, usar `Migration` explícita e manter
`exportSchema = true` com `schemas/` versionado. Nunca usar `fallbackToDestructiveMigration()`
em produção (perde dados do usuário).
