<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="orderSuccess" class="success-banner">{{ t('restocking.orderPlaced') }}</div>
    <div v-if="error" class="error">{{ error }}</div>

    <!-- Budget Card -->
    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        <span class="warehouse-tag">{{ t('restocking.restockingWarehouse') }}: {{ restockingWarehouse }}</span>
      </div>
      <div class="budget-controls">
        <input type="range" min="5000" max="150000" step="1000" v-model.number="budgetLimit" class="budget-slider" />
        <div class="budget-stats">
          <div class="budget-stat">
            <span class="budget-stat-label">{{ t('restocking.budgetLabel') }}</span>
            <span class="budget-stat-value">{{ currencySymbol }}{{ budgetLimit.toLocaleString() }}</span>
          </div>
          <div class="budget-stat">
            <span class="budget-stat-label">{{ t('restocking.selectedTotal') }}</span>
            <span :class="['budget-stat-value', { 'over-budget-value': isOverBudget }]">
              {{ currencySymbol }}{{ Math.round(runningTotal).toLocaleString() }}
            </span>
          </div>
          <div class="budget-stat">
            <span class="budget-stat-label">{{ t('restocking.remaining') }}</span>
            <span :class="['budget-stat-value', { 'over-budget-value': isOverBudget }]">
              {{ isOverBudget ? '-' : '' }}{{ currencySymbol }}{{ Math.abs(Math.round(budgetLimit - runningTotal)).toLocaleString() }}
            </span>
          </div>
        </div>
        <div v-if="isOverBudget" class="over-budget-alert">{{ t('restocking.overBudget') }}</div>
      </div>
    </div>

    <!-- Checklist Card -->
    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('demand.demandForecasts') }} ({{ sortedForecasts.length }})</h3>
        <div class="header-actions">
          <button class="btn-secondary" @click="selectAll">{{ t('restocking.selectAll') }}</button>
          <button class="btn-secondary" @click="deselectAll">{{ t('restocking.deselectAll') }}</button>
        </div>
      </div>
      <div v-if="sortedForecasts.length === 0" class="loading">{{ t('restocking.noItems') }}</div>
      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th style="width:40px"></th>
              <th>{{ t('restocking.columns.item') }}</th>
              <th>{{ t('restocking.columns.sku') }}</th>
              <th>{{ t('restocking.columns.trend') }}</th>
              <th>{{ t('restocking.columns.forecastedQty') }}</th>
              <th>{{ t('restocking.columns.unitCost') }}</th>
              <th>{{ t('restocking.columns.lineTotal') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="forecast in sortedForecasts" :key="forecast.id" :class="{ 'row-unchecked': !checkedItems[forecast.id] }">
              <td><input type="checkbox" v-model="checkedItems[forecast.id]" class="item-checkbox" /></td>
              <td>{{ forecast.item_name }}</td>
              <td><strong>{{ forecast.item_sku }}</strong></td>
              <td><span :class="['badge', forecast.trend]">{{ t(`trends.${forecast.trend}`) }}</span></td>
              <td>{{ forecast.forecasted_demand.toLocaleString() }}</td>
              <td>{{ currencySymbol }}{{ forecast.unit_cost.toFixed(2) }}</td>
              <td><strong>{{ currencySymbol }}{{ Math.round(forecast.forecasted_demand * forecast.unit_cost).toLocaleString() }}</strong></td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Place Order footer -->
      <div class="order-footer">
        <div class="order-summary">
          <span class="order-summary-text">
            {{ checkedCount }} items selected &mdash; {{ currencySymbol }}{{ Math.round(runningTotal).toLocaleString() }}
          </span>
        </div>
        <button
          class="btn-primary"
          :disabled="isOverBudget || isSubmitting || checkedCount === 0"
          @click="placeOrder"
        >
          {{ isSubmitting ? t('common.loading') : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

const TREND_ORDER = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')
    const { selectedLocation } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const allForecasts = ref([])
    const checkedItems = ref({})
    const budgetLimit = ref(100000)
    const isSubmitting = ref(false)
    const orderSuccess = ref(false)
    const successTimer = ref(null)

    const sortedForecasts = computed(() => {
      return [...allForecasts.value].sort(
        (a, b) => TREND_ORDER[a.trend] - TREND_ORDER[b.trend]
      )
    })

    const runningTotal = computed(() => {
      return sortedForecasts.value
        .filter(f => checkedItems.value[f.id])
        .reduce((sum, f) => sum + f.forecasted_demand * f.unit_cost, 0)
    })

    const isOverBudget = computed(() => runningTotal.value > budgetLimit.value)

    const checkedCount = computed(() =>
      Object.values(checkedItems.value).filter(Boolean).length
    )

    const restockingWarehouse = computed(() =>
      selectedLocation.value !== 'all' ? selectedLocation.value : 'San Francisco'
    )

    const selectAll = () => {
      allForecasts.value.forEach(f => { checkedItems.value[f.id] = true })
    }

    const deselectAll = () => {
      allForecasts.value.forEach(f => { checkedItems.value[f.id] = false })
    }

    const placeOrder = async () => {
      if (isOverBudget.value || isSubmitting.value || checkedCount.value === 0) return
      isSubmitting.value = true
      try {
        const items = sortedForecasts.value
          .filter(f => checkedItems.value[f.id])
          .map(f => ({
            item_sku: f.item_sku,
            item_name: f.item_name,
            quantity: f.forecasted_demand,
            unit_cost: f.unit_cost,
            line_total: parseFloat((f.forecasted_demand * f.unit_cost).toFixed(2))
          }))
        await api.createRestockingOrder({ warehouse: restockingWarehouse.value, items })
        orderSuccess.value = true
        if (successTimer.value) clearTimeout(successTimer.value)
        successTimer.value = setTimeout(() => { orderSuccess.value = false }, 5000)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        isSubmitting.value = false
      }
    }

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getDemandForecasts()
        allForecasts.value = data
        const initialChecked = {}
        data.forEach(f => { initialChecked[f.id] = true })
        checkedItems.value = initialChecked
      } catch (err) {
        error.value = 'Failed to load demand forecasts'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    onMounted(() => loadForecasts())

    return {
      t,
      currencySymbol,
      loading,
      error,
      sortedForecasts,
      checkedItems,
      budgetLimit,
      isSubmitting,
      orderSuccess,
      runningTotal,
      isOverBudget,
      checkedCount,
      restockingWarehouse,
      selectAll,
      deselectAll,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card .card-header {
  align-items: center;
}
.warehouse-tag {
  font-size: 0.875rem;
  color: #64748b;
}
.budget-controls {
  padding: 0.5rem 0;
}
.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 1rem;
}
.budget-stats {
  display: flex;
  gap: 2rem;
  margin-bottom: 0.5rem;
}
.budget-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}
.budget-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}
.budget-stat-value {
  font-size: 1.375rem;
  font-weight: 700;
  color: #0f172a;
}
.over-budget-value {
  color: #dc2626;
}
.over-budget-alert {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 0.625rem 1rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
  margin-top: 0.5rem;
}
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-size: 0.938rem;
  font-weight: 500;
}
.header-actions {
  display: flex;
  gap: 0.5rem;
}
.btn-secondary {
  background: white;
  border: 1px solid #e2e8f0;
  color: #475569;
  padding: 0.375rem 0.875rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s ease;
}
.btn-secondary:hover {
  background: #f1f5f9;
  border-color: #cbd5e1;
}
.item-checkbox {
  width: 16px;
  height: 16px;
  accent-color: #2563eb;
  cursor: pointer;
}
.row-unchecked {
  opacity: 0.45;
}
.order-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}
.order-summary-text {
  font-size: 0.938rem;
  color: #475569;
  font-weight: 500;
}
.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s ease;
}
.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}
.btn-primary:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}
</style>
