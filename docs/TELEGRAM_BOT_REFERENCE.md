# Исходники альфа-версии Telegram-бота для переноса

Зафиксировано 19 октября 2026 года из предоставленной локальной папки cashback_bot. Это справочная копия выбранных файлов, а не готовый Android-модуль. Зависимости, репозитории и прочие части проекта здесь не включены. Секреты и конфигурация окружения не копировались. Существующие тесты обнаружены, но при ознакомлении не запускались.

## cashback/MonthlyCashback.kt

```kotlin
package com.example.demo.cashback

import com.example.demo.bank.Bank
import com.example.demo.category.Category
import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.EnumType
import jakarta.persistence.Enumerated
import jakarta.persistence.FetchType
import jakarta.persistence.GeneratedValue
import jakarta.persistence.GenerationType
import jakarta.persistence.Id
import jakarta.persistence.JoinColumn
import jakarta.persistence.ManyToOne
import jakarta.persistence.Table
import jakarta.persistence.UniqueConstraint
import java.math.BigDecimal
import java.time.LocalDate

@Entity
@Table(
    name = "monthly_cashback",
    uniqueConstraints = [
        UniqueConstraint(columnNames = ["chat_id", "bank_id", "category_id", "month_date"]),
    ],
)
class MonthlyCashback(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    @Column(name = "chat_id", nullable = false)
    val chatId: String = "",
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "bank_id", nullable = false)
    val bank: Bank? = null,
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "category_id", nullable = false)
    val category: Category? = null,
    @Column(name = "month_date", nullable = false)
    val month: LocalDate = LocalDate.of(1970, 1, 1),
    @Column(name = "percent", nullable = false, precision = 5, scale = 2)
    val percent: BigDecimal = BigDecimal.ZERO,
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 16)
    val status: CashbackStatus = CashbackStatus.ACTIVE,
) {
    // JPA requires a no-arg constructor.
    protected constructor() : this(0, "", null, null, LocalDate.of(1970, 1, 1), BigDecimal.ZERO, CashbackStatus.ACTIVE)
}

```

## cashback/CashbackStatus.kt

```kotlin
package com.example.demo.cashback

enum class CashbackStatus {
    DRAFT,
    ACTIVE,
}

```

## cashback/MonthlyCashbackService.kt

```kotlin
package com.example.demo.cashback

import com.example.demo.bank.BankRepository
import com.example.demo.category.CategoryRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.math.BigDecimal
import java.time.LocalDate

interface MonthlyCashbackOperations {
    fun list(
        chatId: String,
        month: LocalDate,
    ): List<MonthlyCashback>

    fun hasAnyForMonth(
        chatId: String,
        month: LocalDate,
    ): Boolean

    fun hasAnyForBank(
        chatId: String,
        bankId: Long,
    ): Boolean

    fun hasAnyForCategory(
        chatId: String,
        categoryId: Long,
    ): Boolean

    fun create(
        chatId: String,
        bankId: Long,
        categoryId: Long,
        month: LocalDate,
        percent: BigDecimal,
    ): MonthlyCashbackCreateResult

    fun update(
        chatId: String,
        id: Long,
        percent: BigDecimal,
    ): MonthlyCashbackUpdateResult

    fun delete(
        chatId: String,
        id: Long,
    ): MonthlyCashbackDeleteResult

    fun deleteByIds(
        chatId: String,
        ids: List<Long>,
    ): MonthlyCashbackDeleteManyResult

    fun deleteByBank(
        chatId: String,
        bankId: Long,
    )

    fun deleteDraftsByBank(
        chatId: String,
        bankId: Long,
        month: LocalDate,
    ): Long

    fun deleteByCategory(
        chatId: String,
        categoryId: Long,
    )

    fun copyToNextMonth(
        chatId: String,
        bankId: Long,
        fromMonth: LocalDate,
    ): MonthlyCashbackCopyResult

    fun updateStatus(
        chatId: String,
        ids: List<Long>,
        status: CashbackStatus,
    ): MonthlyCashbackStatusUpdateResult
}

@Service
class MonthlyCashbackService(
    private val repository: MonthlyCashbackRepository,
    private val bankRepository: BankRepository,
    private val categoryRepository: CategoryRepository,
) : MonthlyCashbackOperations {
    override fun list(
        chatId: String,
        month: LocalDate,
    ): List<MonthlyCashback> = repository.findByChatIdAndMonthOrderByCategory_NameAsc(chatId, month.normalizeMonth())

    override fun hasAnyForMonth(
        chatId: String,
        month: LocalDate,
    ): Boolean = repository.existsByChatIdAndMonth(chatId, month.normalizeMonth())

    override fun hasAnyForBank(
        chatId: String,
        bankId: Long,
    ): Boolean = repository.existsByChatIdAndBank_Id(chatId, bankId)

    override fun hasAnyForCategory(
        chatId: String,
        categoryId: Long,
    ): Boolean = repository.existsByChatIdAndCategory_Id(chatId, categoryId)

    override fun create(
        chatId: String,
        bankId: Long,
        categoryId: Long,
        month: LocalDate,
        percent: BigDecimal,
    ): MonthlyCashbackCreateResult {
        val normalizedMonth = month.normalizeMonth()
        if (!CashbackPercentValidator.isValid(percent)) {
            return MonthlyCashbackCreateResult.InvalidPercent
        }
        val bank = bankRepository.findByIdAndChatId(bankId, chatId) ?: return MonthlyCashbackCreateResult.BankNotFound
        val category = categoryRepository.findByIdAndChatId(categoryId, chatId) ?: return MonthlyCashbackCreateResult.CategoryNotFound
        val existing = repository.findByChatIdAndBank_IdAndCategory_IdAndMonth(chatId, bankId, categoryId, normalizedMonth)
        if (existing != null) {
            return if (existing.percent.compareTo(percent) == 0) {
                MonthlyCashbackCreateResult.SameValue(existing)
            } else {
                MonthlyCashbackCreateResult.ConfirmUpdate(existing, percent)
            }
        }
        val saved =
            repository.save(
                MonthlyCashback(
                    chatId = chatId,
                    bank = bank,
                    category = category,
                    month = normalizedMonth,
                    percent = percent,
                    status = CashbackStatus.DRAFT,
                ),
            )
        return MonthlyCashbackCreateResult.Created(saved)
    }

    override fun update(
        chatId: String,
        id: Long,
        percent: BigDecimal,
    ): MonthlyCashbackUpdateResult {
        if (!CashbackPercentValidator.isValid(percent)) {
            return MonthlyCashbackUpdateResult.InvalidPercent
        }
        val cashback = repository.findByIdAndChatId(id, chatId) ?: return MonthlyCashbackUpdateResult.NotFound
        val updated =
            MonthlyCashback(
                id = cashback.id,
                chatId = cashback.chatId,
                bank = cashback.bank,
                category = cashback.category,
                month = cashback.month,
                percent = percent,
                status = cashback.status,
            )
        return MonthlyCashbackUpdateResult.Updated(repository.save(updated))
    }

    override fun delete(
        chatId: String,
        id: Long,
    ): MonthlyCashbackDeleteResult {
        val cashback = repository.findByIdAndChatId(id, chatId) ?: return MonthlyCashbackDeleteResult.NotFound
        repository.delete(cashback)
        return MonthlyCashbackDeleteResult.Deleted(cashback)
    }

    @Transactional
    override fun deleteByIds(
        chatId: String,
        ids: List<Long>,
    ): MonthlyCashbackDeleteManyResult {
        val uniqueIds = ids.distinct()
        val entries = repository.findByIdInAndChatId(uniqueIds, chatId)
        if (entries.isEmpty()) {
            return MonthlyCashbackDeleteManyResult.NotFound
        }
        repository.deleteAll(entries)
        val foundIds = entries.map { it.id }.toSet()
        val missing = uniqueIds.filterNot { foundIds.contains(it) }
        return MonthlyCashbackDeleteManyResult.Deleted(entries.size, missing)
    }

    @Transactional
    override fun deleteByBank(
        chatId: String,
        bankId: Long,
    ) {
        repository.deleteByChatIdAndBank_Id(chatId, bankId)
    }

    @Transactional
    override fun deleteDraftsByBank(
        chatId: String,
        bankId: Long,
        month: LocalDate,
    ): Long = repository.deleteByChatIdAndBank_IdAndMonthAndStatus(chatId, bankId, month.normalizeMonth(), CashbackStatus.DRAFT)

    @Transactional
    override fun deleteByCategory(
        chatId: String,
        categoryId: Long,
    ) {
        repository.deleteByChatIdAndCategory_Id(chatId, categoryId)
    }

    override fun copyToNextMonth(
        chatId: String,
        bankId: Long,
        fromMonth: LocalDate,
    ): MonthlyCashbackCopyResult {
        val bank = bankRepository.findByIdAndChatId(bankId, chatId) ?: return MonthlyCashbackCopyResult.BankNotFound
        val normalizedFrom = fromMonth.normalizeMonth()
        val normalizedTo = normalizedFrom.plusMonths(1)
        val source = repository.findByChatIdAndBank_IdAndMonth(chatId, bankId, normalizedFrom)
        if (source.isEmpty()) {
            return MonthlyCashbackCopyResult.NothingToCopy
        }
        val existingTarget =
            repository
                .findByChatIdAndBank_IdAndMonth(chatId, bankId, normalizedTo)
                .mapNotNull { it.category?.id }
                .toSet()
        var created = 0
        source.forEach { entry ->
            val categoryId = entry.category?.id ?: return@forEach
            if (existingTarget.contains(categoryId)) {
                return@forEach
            }
            val category = categoryRepository.findByIdAndChatId(categoryId, chatId) ?: return@forEach
            repository.save(
                MonthlyCashback(
                    chatId = chatId,
                    bank = bank,
                    category = category,
                    month = normalizedTo,
                    percent = entry.percent,
                    status = CashbackStatus.DRAFT,
                ),
            )
            created += 1
        }
        return MonthlyCashbackCopyResult.Copied(created)
    }

    override fun updateStatus(
        chatId: String,
        ids: List<Long>,
        status: CashbackStatus,
    ): MonthlyCashbackStatusUpdateResult {
        val uniqueIds = ids.distinct()
        val entries = repository.findByIdInAndChatId(uniqueIds, chatId)
        if (entries.isEmpty()) {
            return MonthlyCashbackStatusUpdateResult.NotFound
        }
        val updated =
            entries.map { entry ->
                if (entry.status == status) {
                    entry
                } else {
                    MonthlyCashback(
                        id = entry.id,
                        chatId = entry.chatId,
                        bank = entry.bank,
                        category = entry.category,
                        month = entry.month,
                        percent = entry.percent,
                        status = status,
                    )
                }
            }
        repository.saveAll(updated)
        val foundIds = entries.map { it.id }.toSet()
        val missing = uniqueIds.filterNot { foundIds.contains(it) }
        return MonthlyCashbackStatusUpdateResult.Updated(updated.size, missing)
    }
}

sealed class MonthlyCashbackCreateResult {
    data class Created(
        val cashback: MonthlyCashback,
    ) : MonthlyCashbackCreateResult()

    data class SameValue(
        val cashback: MonthlyCashback,
    ) : MonthlyCashbackCreateResult()

    data class ConfirmUpdate(
        val cashback: MonthlyCashback,
        val newPercent: BigDecimal,
    ) : MonthlyCashbackCreateResult()

    data object BankNotFound : MonthlyCashbackCreateResult()

    data object CategoryNotFound : MonthlyCashbackCreateResult()

    data object InvalidPercent : MonthlyCashbackCreateResult()
}

sealed class MonthlyCashbackUpdateResult {
    data class Updated(
        val cashback: MonthlyCashback,
    ) : MonthlyCashbackUpdateResult()

    data object NotFound : MonthlyCashbackUpdateResult()

    data object InvalidPercent : MonthlyCashbackUpdateResult()
}

sealed class MonthlyCashbackDeleteResult {
    data class Deleted(
        val cashback: MonthlyCashback,
    ) : MonthlyCashbackDeleteResult()

    data object NotFound : MonthlyCashbackDeleteResult()
}

sealed class MonthlyCashbackDeleteManyResult {
    data class Deleted(
        val deletedCount: Int,
        val missingIds: List<Long>,
    ) : MonthlyCashbackDeleteManyResult()

    data object NotFound : MonthlyCashbackDeleteManyResult()
}

sealed class MonthlyCashbackCopyResult {
    data class Copied(
        val createdCount: Int,
    ) : MonthlyCashbackCopyResult()

    data object NothingToCopy : MonthlyCashbackCopyResult()

    data object BankNotFound : MonthlyCashbackCopyResult()
}

sealed class MonthlyCashbackStatusUpdateResult {
    data class Updated(
        val updatedCount: Int,
        val missingIds: List<Long>,
    ) : MonthlyCashbackStatusUpdateResult()

    data object NotFound : MonthlyCashbackStatusUpdateResult()
}

private fun LocalDate.normalizeMonth(): LocalDate = withDayOfMonth(1)

object CashbackPercentValidator {
    private val maxPercent = BigDecimal("100")

    fun isValid(value: BigDecimal): Boolean {
        if (value < BigDecimal.ZERO) return false
        return value <= maxPercent
    }
}

```

## category/CategoryService.kt

```kotlin
package com.example.demo.category

import com.example.demo.domain.crud.CrudOperations
import org.springframework.stereotype.Service

interface CategoryOperations : CrudOperations<String, Category, CategoryCreateResult, CategoryUpdateResult, CategoryDeleteResult>

@Service
class CategoryService(
    private val repository: CategoryRepository,
) : CategoryOperations {
    override fun list(context: String): List<Category> {
        ensureDefaults(context)
        return repository.findAllByChatIdOrderByNameAsc(context)
    }

    override fun find(
        context: String,
        id: Long,
    ): Category? {
        ensureDefaults(context)
        return repository.findByIdAndChatId(id, context)
    }

    override fun create(
        context: String,
        name: String,
    ): CategoryCreateResult {
        ensureDefaults(context)
        val trimmed = name.trim()
        if (trimmed.isEmpty()) {
            return CategoryCreateResult.BlankName
        }
        if (repository.findByChatIdAndName(context, trimmed) != null) {
            return CategoryCreateResult.Duplicate
        }
        val category = repository.save(Category(chatId = context, name = trimmed))
        return CategoryCreateResult.Created(category)
    }

    override fun update(
        context: String,
        id: Long,
        name: String,
    ): CategoryUpdateResult {
        ensureDefaults(context)
        val trimmed = name.trim()
        if (trimmed.isEmpty()) {
            return CategoryUpdateResult.BlankName
        }
        val category = repository.findByIdAndChatId(id, context) ?: return CategoryUpdateResult.NotFound
        if (category.name == trimmed) {
            return CategoryUpdateResult.Updated(category)
        }
        if (repository.findByChatIdAndName(context, trimmed) != null) {
            return CategoryUpdateResult.Duplicate
        }
        category.name = trimmed
        return CategoryUpdateResult.Updated(repository.save(category))
    }

    override fun delete(
        context: String,
        id: Long,
    ): CategoryDeleteResult {
        ensureDefaults(context)
        val category = repository.findByIdAndChatId(id, context) ?: return CategoryDeleteResult.NotFound
        repository.delete(category)
        return CategoryDeleteResult.Deleted(category)
    }

    private fun ensureDefaults(chatId: String) {
        if (repository.countByChatId(chatId) > 0) return
        val defaults = DEFAULT_CATEGORY_NAMES.map { Category(chatId = chatId, name = it) }
        repository.saveAll(defaults)
    }
}

private val DEFAULT_CATEGORY_NAMES =
    listOf(
        "Вкусвилл.Тинькофф",
        "Маркетплейсы",
        "Супермаркеты",
        "Рестораны",
        "Фастфуд",
        "Яндекс.Еда",
        "Лавка",
        "Здоровье(клиники)",
        "Аптеки",
        "Оптика и ортопедия",
        "Еаптека",
        "Косметика",
        "Красота",
        "DNS",
        "Шоппинг Тинькофф",
        "Электроника",
        "Химчистка",
        "Цифровой контент",
        "Яндекс.Заправки",
        "АЗС",
        "Одежда и обувь",
        "ЖКХ",
        "Детские товары",
        "Ветклиники",
        "Зоотовары",
        "Аксессуары",
        "Сувениры",
        "Фитнес",
        "Развлечения",
        "Хобби",
        "Мебель",
        "Выставки и музеи",
        "Концерты.Тинькофф",
        "Театры и кино",
        "Строительные",
        "Цветы",
        "Автоуслуги",
        "Ювелирные изделия",
        "Книги",
        "Спорттовары",
        "Образование",
        "Такси",
        "ЖД",
        "Городской транспорт",
        "Все остальное",
        "Tinkoff.Premium",
    )

sealed class CategoryCreateResult {
    data class Created(
        val category: Category,
    ) : CategoryCreateResult()

    data object Duplicate : CategoryCreateResult()

    data object BlankName : CategoryCreateResult()
}

sealed class CategoryUpdateResult {
    data class Updated(
        val category: Category,
    ) : CategoryUpdateResult()

    data object Duplicate : CategoryUpdateResult()

    data object BlankName : CategoryUpdateResult()

    data object NotFound : CategoryUpdateResult()
}

sealed class CategoryDeleteResult {
    data class Deleted(
        val category: Category,
    ) : CategoryDeleteResult()

    data object NotFound : CategoryDeleteResult()
}

```

## Выделение лучших процентов в текущем боте

Фрагмент BotFlowCoordinator.kt. Только подсветка максимума по категории; не алгоритм совместного подбора с лимитами банков.

```kotlin
    private fun optimalCashbackIds(list: List<com.example.demo.cashback.MonthlyCashback>): Set<Long> {
        if (list.isEmpty()) return emptySet()
        val result = mutableSetOf<Long>()
        val grouped = list.groupBy { it.category?.id }
        grouped.values.forEach { entries ->
            if (entries.isEmpty()) return@forEach
            val maxPercent = entries.maxWithOrNull { a, b -> a.percent.compareTo(b.percent) }?.percent ?: return@forEach
            entries.filter { it.percent.compareTo(maxPercent) == 0 }.forEach { result.add(it.id) }
        }
        return result
    }

```
