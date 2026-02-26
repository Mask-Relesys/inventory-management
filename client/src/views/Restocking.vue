<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Auto-select items to restock within your budget</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success banner -->
      <div v-if="orderSuccess" class="success-banner">
        <span>Order {{ orderSuccess }} placed successfully</span>
        <button class="dismiss-btn" @click="orderSuccess = null">Dismiss</button>
      </div>

      <!-- Order error banner -->
      <div v-if="orderError" class="order-error-banner">
        <span>{{ orderError }}</span>
        <button class="dismiss-btn dismiss-btn--error" @click="orderError = null">Dismiss</button>
      </div>

      <!-- Budget card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-body">
          <input
            type="range"
            class="budget-slider"
            v-model.number="budget"
            min="0"
            max="100000"
            step="1000"
          />
          <div class="budget-display">
            <span>Budget: {{ currencySymbol }}{{ budget.toLocaleString() }}</span>
            <span class="budget-separator">|</span>
            <span>Selected: {{ currencySymbol }}{{ totalCost.toLocaleString() }}</span>
            <span class="budget-separator">|</span>
            <span :class="remainingBudget < 0 ? 'budget-over' : 'budget-ok'">
              Remaining: {{ currencySymbol }}{{ remainingBudget.toLocaleString() }}
            </span>
          </div>
        </div>
      </div>

      <!-- Recommended items card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ sortedForecasts.length }})</h3>
        </div>
        <div class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-name">Item Name</th>
                <th class="col-trend">Trend</th>
                <th class="col-qty">Forecasted Qty</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-total">Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="forecast in sortedForecasts"
                :key="forecast.id"
                :class="{ 'row-selected': isSelected(forecast) }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="isSelected(forecast)"
                    @change="toggleItem(forecast)"
                  />
                </td>
                <td class="col-name">{{ forecast.item_name }}</td>
                <td class="col-trend">
                  <span :class="['badge', forecast.trend]">{{ forecast.trend }}</span>
                </td>
                <td class="col-qty">{{ forecast.forecasted_demand.toLocaleString() }}</td>
                <td class="col-cost">{{ currencySymbol }}{{ forecast.unit_cost.toLocaleString() }}</td>
                <td class="col-total">
                  <strong>{{ currencySymbol }}{{ lineTotal(forecast).toLocaleString() }}</strong>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="card-footer">
          <button
            class="place-order-btn"
            :disabled="selectedItems.length === 0 || placing"
            @click="placeOrder"
          >
            {{ placing ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

// Trend priority for sorting: increasing first, then stable, then decreasing
const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    // --- Data loading state ---
    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])

    // --- Budget and selection state ---
    const budget = ref(50000)
    // manualOverrides: id -> boolean (true = checked, false = unchecked)
    // Entries here override the auto-selection computed from the budget slider
    const manualOverrides = ref({})

    // --- Order state ---
    const placing = ref(false)
    const orderSuccess = ref(null)
    const orderError = ref(null)

    // --- Data loading ---
    const loadForecasts = async () => {
      try {
        loading.value = true
        error.value = null
        forecasts.value = await api.getDemandForecasts()
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
        console.error('Restocking load error:', err)
      } finally {
        loading.value = false
      }
    }

    onMounted(loadForecasts)

    // --- Computed: line total helper ---
    const lineTotal = (forecast) => forecast.forecasted_demand * forecast.unit_cost

    // --- Computed: sorted forecasts ---
    // Sort by trend priority (increasing > stable > decreasing), then by line total descending
    const sortedForecasts = computed(() => {
      return [...forecasts.value].sort((a, b) => {
        const trendDiff = (TREND_PRIORITY[a.trend] ?? 99) - (TREND_PRIORITY[b.trend] ?? 99)
        if (trendDiff !== 0) return trendDiff
        return lineTotal(b) - lineTotal(a)
      })
    })

    // --- Computed: auto-selected IDs from greedy budget fill ---
    // Returns a Set of ids for items that would be checked purely based on budget
    const autoSelectedIds = computed(() => {
      const selected = new Set()
      let remaining = budget.value
      for (const forecast of sortedForecasts.value) {
        const cost = lineTotal(forecast)
        if (cost <= remaining) {
          selected.add(forecast.id)
          remaining -= cost
        }
      }
      return selected
    })

    // --- Selection logic ---
    // If item has a manual override, use that; otherwise fall back to auto-selection
    const isSelected = (forecast) => {
      if (forecast.id in manualOverrides.value) {
        return manualOverrides.value[forecast.id]
      }
      return autoSelectedIds.value.has(forecast.id)
    }

    // Toggling an item sets a manual override, flipping its current state
    const toggleItem = (forecast) => {
      // Create a new object to ensure reactivity triggers
      manualOverrides.value = {
        ...manualOverrides.value,
        [forecast.id]: !isSelected(forecast)
      }
    }

    // When the budget slider moves, clear all manual overrides so auto-selection takes over
    watch(budget, () => {
      manualOverrides.value = {}
    })

    // --- Computed: derived cost values ---
    const totalCost = computed(() => {
      return sortedForecasts.value.reduce((sum, forecast) => {
        return isSelected(forecast) ? sum + lineTotal(forecast) : sum
      }, 0)
    })

    const remainingBudget = computed(() => budget.value - totalCost.value)

    // --- Computed: items to submit ---
    const selectedItems = computed(() =>
      sortedForecasts.value.filter(forecast => isSelected(forecast))
    )

    // --- Place order ---
    const placeOrder = async () => {
      if (selectedItems.value.length === 0) return
      placing.value = true
      orderError.value = null
      orderSuccess.value = null
      try {
        const items = selectedItems.value.map(i => ({
          sku: i.item_sku,
          name: i.item_name,
          quantity: i.forecasted_demand,
          unit_price: i.unit_cost
        }))
        const result = await api.submitRestockingOrder(items)
        orderSuccess.value = result.order_number
        // Clear manual overrides so selection resets to auto after order
        manualOverrides.value = {}
      } catch (err) {
        orderError.value = 'Failed to place order: ' + err.message
        console.error('Place order error:', err)
      } finally {
        placing.value = false
      }
    }

    return {
      t,
      currencySymbol,
      loading,
      error,
      budget,
      sortedForecasts,
      lineTotal,
      isSelected,
      toggleItem,
      totalCost,
      remainingBudget,
      selectedItems,
      placing,
      orderSuccess,
      orderError,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* Budget card */
.budget-body {
  padding: 1.25rem 1.5rem 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-display {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.9375rem;
  font-weight: 500;
  color: #0f172a;
  flex-wrap: wrap;
}

.budget-separator {
  color: #cbd5e1;
  font-weight: 400;
}

.budget-ok {
  color: #16a34a;
}

.budget-over {
  color: #dc2626;
}

/* Table */
.restocking-table {
  table-layout: fixed;
  width: 100%;
}

.col-check {
  width: 40px;
}

.col-name {
  /* flexible, takes remaining space */
}

.col-trend {
  width: 110px;
}

.col-qty {
  width: 130px;
  text-align: right;
}

.col-cost {
  width: 110px;
  text-align: right;
}

.col-total {
  width: 130px;
  text-align: right;
}

.restocking-table th.col-qty,
.restocking-table th.col-cost,
.restocking-table th.col-total {
  text-align: right;
}

.row-selected {
  background-color: #eff6ff;
}

/* Card footer with Place Order button */
.card-footer {
  padding: 1rem 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
}

.place-order-btn {
  background-color: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 6px;
  padding: 0.625rem 1.5rem;
  font-size: 0.9375rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.15s;
}

.place-order-btn:hover:not(:disabled) {
  background-color: #1d4ed8;
}

.place-order-btn:disabled {
  background-color: #94a3b8;
  cursor: not-allowed;
}

/* Success banner */
.success-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background-color: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  margin-bottom: 1.25rem;
  font-weight: 500;
}

/* Order error banner */
.order-error-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background-color: #fee2e2;
  border: 1px solid #fca5a5;
  color: #991b1b;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  margin-bottom: 1.25rem;
  font-weight: 500;
}

.dismiss-btn {
  background: none;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 4px;
  padding: 0.25rem 0.75rem;
  font-size: 0.8125rem;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.15s;
}

.dismiss-btn:hover {
  background-color: #a7f3d0;
}

.dismiss-btn--error {
  border-color: #fca5a5;
  color: #991b1b;
}

.dismiss-btn--error:hover {
  background-color: #fecaca;
}
</style>
