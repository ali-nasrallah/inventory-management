<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Slider Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.budgetLabel') }}:
            <span class="budget-chip">{{ formatCurrency(budget, currentCurrency) }}</span>
          </h3>
        </div>
        <div class="slider-row">
          <span class="slider-label">{{ t('restocking.minLabel') }}: {{ formatCurrency(1000, currentCurrency) }}</span>
          <input
            type="range"
            :min="1000"
            :max="100000"
            :step="1000"
            v-model.number="budget"
            class="budget-slider"
          />
          <span class="slider-label">{{ t('restocking.maxLabel') }}: {{ formatCurrency(100000, currentCurrency) }}</span>
        </div>
        <p class="helper-text">{{ t('restocking.helperText') }}</p>
      </div>

      <!-- Summary Stats -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.itemCount') }}</div>
          <div class="stat-value">{{ itemCount }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.totalCost') }}</div>
          <div class="stat-value">{{ formatCurrencyWithDecimals(totalCost, currentCurrency, 2) }}</div>
        </div>
        <div :class="['stat-card', remaining >= 0 ? 'warning' : 'danger']">
          <div class="stat-label">{{ t('restocking.remaining') }}</div>
          <div class="stat-value">{{ formatCurrencyWithDecimals(remaining, currentCurrency, 2) }}</div>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.recommendationsTitle') }} ({{ recommendations.length }})
          </h3>
        </div>

        <p v-if="recommendations.length === 0" class="no-recommendations">
          {{ t('restocking.noRecommendations') }}
        </p>
        <div v-else class="table-container">
          <table class="orders-table">
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.name') }}</th>
                <th>{{ t('restocking.table.warehouse') }}</th>
                <th>{{ t('restocking.table.category') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.qty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="r in recommendations" :key="r.id">
                <td><strong>{{ r.item_sku }}</strong></td>
                <td>{{ translateProductName(r.item_name) }}</td>
                <td>{{ r.warehouse }}</td>
                <td>{{ r.category }}</td>
                <td>
                  <span :class="['badge', r.trend]">
                    {{ t(`trends.${r.trend}`) }}
                  </span>
                </td>
                <td>{{ r.forecasted_demand }}</td>
                <td>{{ formatCurrencyWithDecimals(r.unit_cost, currentCurrency, 2) }}</td>
                <td><strong>{{ formatCurrencyWithDecimals(r.cost, currentCurrency, 2) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Action Row -->
      <div class="action-row">
        <button
          class="btn-primary"
          :disabled="recommendations.length === 0 || submitState === 'submitting'"
          @click="placeOrder"
        >
          {{ submitState === 'submitting' ? t('restocking.submitting') : t('restocking.placeOrder') }}
        </button>
        <span v-if="submitState === 'success'" class="submit-success">
          {{ t('restocking.submitSuccess', { orderNumber: lastOrderNumber }) }}
        </span>
        <span v-if="submitState === 'error'" class="submit-error">
          {{ t('restocking.submitError') }}
        </span>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'
import { formatCurrency, formatCurrencyWithDecimals } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const loading = ref(true)
    const error = ref(null)
    const allForecasts = ref([])
    const inventoryItems = ref([])
    const budget = ref(20000)
    const submitState = ref('idle') // 'idle' | 'submitting' | 'success' | 'error'
    const lastOrderNumber = ref(null)

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({ warehouse: filters.warehouse, category: filters.category })
        ])
        allForecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    onMounted(loadData)
    watch([selectedLocation, selectedCategory], loadData)

    // Join forecasts with inventory to build candidates
    const candidates = computed(() => {
      const inventoryMap = {}
      for (const item of inventoryItems.value) {
        inventoryMap[item.sku] = item
      }

      return allForecasts.value
        .filter(f => inventoryMap[f.item_sku])
        .map(f => {
          const inv = inventoryMap[f.item_sku]
          const cost = f.forecasted_demand * inv.unit_cost
          const changePercent = inv.unit_cost > 0
            ? ((f.forecasted_demand - f.current_demand) / f.current_demand) * 100
            : 0
          return {
            ...f,
            unit_cost: inv.unit_cost,
            warehouse: inv.warehouse,
            category: inv.category,
            cost,
            changePercent
          }
        })
    })

    // Sort: increasing (0) > stable (1) > decreasing (2), tie-break by changePercent descending
    const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }
    const rankedCandidates = computed(() => {
      return [...candidates.value].sort((a, b) => {
        const trendDiff = (trendOrder[a.trend] ?? 3) - (trendOrder[b.trend] ?? 3)
        if (trendDiff !== 0) return trendDiff
        return b.changePercent - a.changePercent
      })
    })

    // Greedy skip-don't-stop: include each item that fits within remaining budget
    const recommendations = computed(() => {
      let running = 0
      const result = []
      for (const candidate of rankedCandidates.value) {
        if (running + candidate.cost <= budget.value) {
          running += candidate.cost
          result.push(candidate)
        }
        // skip if over budget, but keep checking further items
      }
      return result
    })

    const totalCost = computed(() => recommendations.value.reduce((sum, r) => sum + r.cost, 0))
    const remaining = computed(() => budget.value - totalCost.value)
    const itemCount = computed(() => recommendations.value.length)

    const placeOrder = async () => {
      if (recommendations.value.length === 0) return
      submitState.value = 'submitting'

      const payload = {
        items: recommendations.value.map(r => ({
          sku: r.item_sku,
          name: r.item_name,
          quantity: r.forecasted_demand,
          unit_price: r.unit_cost
        })),
        customer: 'Internal Restock',
        warehouse: selectedLocation.value !== 'all' ? selectedLocation.value : null,
        category: selectedCategory.value !== 'all' ? selectedCategory.value : null
      }

      try {
        const created = await api.createRestockingOrder(payload)
        lastOrderNumber.value = created.order_number
        submitState.value = 'success'
        setTimeout(() => {
          submitState.value = 'idle'
        }, 3000)
      } catch (err) {
        console.error('Failed to submit restocking order:', err)
        submitState.value = 'error'
        setTimeout(() => {
          submitState.value = 'idle'
        }, 3000)
      }
    }

    return {
      t,
      currentCurrency,
      currencySymbol,
      translateProductName,
      formatCurrency,
      formatCurrencyWithDecimals,
      loading,
      error,
      budget,
      recommendations,
      totalCost,
      remaining,
      itemCount,
      submitState,
      lastOrderNumber,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* Budget slider */
.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 0.75rem;
}

.budget-slider {
  flex: 1;
  appearance: none;
  -webkit-appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  transition: background 0.2s;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #0f172a;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.25);
  transition: background 0.2s;
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #0f172a;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.25);
}

.budget-slider:hover::-webkit-slider-thumb {
  background: #334155;
}

.budget-slider:hover::-moz-range-thumb {
  background: #334155;
}

.budget-slider::-webkit-slider-runnable-track {
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
}

.budget-slider::-moz-range-track {
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
}

.helper-text {
  font-size: 0.813rem;
  color: #64748b;
  margin-top: 0.5rem;
}

/* Budget chip */
.budget-chip {
  display: inline-block;
  background: #f1f5f9;
  color: #0f172a;
  font-size: 0.938rem;
  font-weight: 600;
  padding: 0.25rem 0.625rem;
  border-radius: 6px;
  border: 1px solid #e2e8f0;
  margin-left: 0.5rem;
  letter-spacing: 0;
}

/* Recommendations empty state */
.no-recommendations {
  color: #64748b;
  font-size: 0.938rem;
  padding: 1.5rem 0;
  text-align: center;
}

/* Action row */
.action-row {
  display: flex;
  align-items: center;
  gap: 1.25rem;
  margin-bottom: 1.25rem;
}

/* Primary button */
.btn-primary {
  background: #0f172a;
  color: white;
  border: none;
  padding: 0.625rem 1.25rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
  white-space: nowrap;
}

.btn-primary:hover:not(:disabled) {
  background: #1e293b;
}

.btn-primary:disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
}

/* Submit feedback messages */
.submit-success {
  font-size: 0.875rem;
  color: #059669;
  font-weight: 500;
}

.submit-error {
  font-size: 0.875rem;
  color: #dc2626;
  font-weight: 500;
}

/* Table column widths */
.orders-table {
  table-layout: auto;
  width: 100%;
}
</style>
