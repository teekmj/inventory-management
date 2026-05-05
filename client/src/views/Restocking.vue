<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Configure your budget, review demand-driven recommendations, and place restocking orders.</p>
    </div>

    <!-- Success banner -->
    <div v-if="lastOrder" class="success-banner">
      <strong>{{ lastOrder.order_number }}</strong> submitted — estimated delivery in {{ leadTime }} days
      <button class="dismiss-btn" @click="lastOrder = null">Dismiss</button>
    </div>

    <!-- Budget Configuration -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Budget Configuration</h3>
      </div>
      <div class="budget-config">
        <div class="config-row">
          <label class="config-label">Available Budget</label>
          <div class="slider-wrapper">
            <input
              type="range"
              min="1000"
              max="50000"
              step="500"
              v-model.number="budget"
              class="budget-slider"
              :style="sliderFillStyle"
            />
            <div class="slider-labels">
              <span>$1,000</span>
              <span class="budget-value">{{ formatCurrency(budget) }}</span>
              <span>$50,000</span>
            </div>
          </div>
        </div>
        <div class="config-row lead-time-row">
          <label class="config-label">Delivery Lead Time</label>
          <div class="lead-time-input">
            <input
              type="number"
              min="3"
              max="30"
              v-model.number="leadTime"
              class="number-input"
            />
            <span class="days-suffix">days</span>
          </div>
        </div>
      </div>
    </div>

    <!-- Recommendations -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Items</h3>
        <div class="budget-summary">
          <span :class="['budget-status', { 'over-budget': isOverBudget }]">
            {{ formatCurrency(selectedCost) }} / {{ formatCurrency(budget) }}
          </span>
        </div>
      </div>

      <!-- Budget usage bar -->
      <div class="budget-bar-container">
        <div class="budget-bar-label">
          Budget Used: <strong>{{ formatCurrency(selectedCost) }}</strong> / {{ formatCurrency(budget) }}
        </div>
        <div class="budget-bar-track">
          <div
            class="budget-bar-fill"
            :style="{ width: budgetUsagePercent + '%', background: isOverBudget ? '#dc2626' : '#2563eb' }"
          ></div>
        </div>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-sku">SKU</th>
                <th class="col-name">Item Name</th>
                <th class="col-trend">Trend</th>
                <th class="col-num">Current</th>
                <th class="col-num">Forecasted</th>
                <th class="col-num">Gap</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Qty</th>
                <th class="col-cost">Est. Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.item_sku"
                :class="{ 'row-selected': selectedSkus.has(item.item_sku) }"
                @click="toggleItem(item.item_sku)"
                class="restock-row"
              >
                <td class="col-check" @click.stop>
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(item.item_sku)"
                    @change="toggleItem(item.item_sku)"
                    class="row-checkbox"
                  />
                </td>
                <td class="col-sku"><code>{{ item.item_sku }}</code></td>
                <td class="col-name">{{ item.item_name }}</td>
                <td class="col-trend">
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-num">{{ item.current_demand.toLocaleString() }}</td>
                <td class="col-num">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-num gap-cell">{{ item.gap.toLocaleString() }}</td>
                <td class="col-num">{{ formatCurrency(item.unit_cost) }}</td>
                <td class="col-num">{{ item.restock_quantity.toLocaleString() }}</td>
                <td class="col-cost"><strong>{{ formatCurrency(item.estimated_cost) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="actions-row">
          <span class="selection-info" v-if="selectedItems.length > 0">
            {{ selectedItems.length }} item{{ selectedItems.length === 1 ? '' : 's' }} selected
          </span>
          <button
            class="place-order-btn"
            :disabled="selectedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            <span v-if="submitting">Submitting...</span>
            <span v-else-if="selectedItems.length === 0">Place Order</span>
            <span v-else>Place Order ({{ selectedItems.length }} item{{ selectedItems.length === 1 ? '' : 's' }} &middot; {{ formatCurrency(selectedCost) }})</span>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(10000)
    const leadTime = ref(7)
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const lastOrder = ref(null)
    // Map<sku, boolean> — true = user forced ON, false = user forced OFF
    const manualOverrides = ref(new Map())

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        recommendations.value = await api.getRestockingRecommendations()
      } catch (err) {
        error.value = 'Failed to load recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Greedy auto-selection based on budget
    const autoSelected = computed(() => {
      const selected = new Set()
      let remaining = budget.value
      for (const item of recommendations.value) {
        if (item.estimated_cost <= remaining) {
          selected.add(item.item_sku)
          remaining -= item.estimated_cost
        }
      }
      return selected
    })

    // Final selection: manual overrides win, otherwise auto
    const selectedSkus = computed(() => {
      const result = new Set()
      for (const item of recommendations.value) {
        const override = manualOverrides.value.get(item.item_sku)
        if (override === true) {
          result.add(item.item_sku)
        } else if (override === false) {
          // explicitly deselected — skip
        } else if (autoSelected.value.has(item.item_sku)) {
          result.add(item.item_sku)
        }
      }
      return result
    })

    const selectedItems = computed(() =>
      recommendations.value.filter(item => selectedSkus.value.has(item.item_sku))
    )

    const selectedCost = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.estimated_cost, 0)
    )

    const isOverBudget = computed(() => selectedCost.value > budget.value)

    const budgetUsagePercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min(100, (selectedCost.value / budget.value) * 100)
    })

    const sliderFillStyle = computed(() => {
      const pct = ((budget.value - 1000) / (50000 - 1000)) * 100
      return {
        background: `linear-gradient(to right, #2563eb ${pct}%, #e2e8f0 ${pct}%)`
      }
    })

    const toggleItem = (sku) => {
      const currentlySelected = selectedSkus.value.has(sku)
      const newMap = new Map(manualOverrides.value)
      newMap.set(sku, !currentlySelected)
      manualOverrides.value = newMap
    }

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0) return
      submitting.value = true
      error.value = null
      try {
        const payload = {
          items: selectedItems.value.map(item => ({
            item_sku: item.item_sku,
            item_name: item.item_name,
            quantity: item.restock_quantity,
            unit_cost: item.unit_cost,
            estimated_cost: item.estimated_cost
          })),
          total_cost: selectedCost.value,
          lead_time_days: leadTime.value
        }
        const order = await api.submitRestockingOrder(payload)
        lastOrder.value = order
        manualOverrides.value = new Map()
      } catch (err) {
        error.value = 'Failed to submit order'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      budget,
      leadTime,
      recommendations,
      loading,
      error,
      submitting,
      lastOrder,
      autoSelected,
      selectedSkus,
      selectedItems,
      selectedCost,
      isOverBudget,
      budgetUsagePercent,
      sliderFillStyle,
      toggleItem,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 2rem 0;
}

/* Success banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.938rem;
}

.dismiss-btn {
  margin-left: auto;
  background: none;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.25rem 0.75rem;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.813rem;
  font-weight: 600;
}

.dismiss-btn:hover {
  background: #a7f3d0;
}

/* Budget config */
.budget-config {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.config-row {
  display: flex;
  align-items: flex-start;
  gap: 1.5rem;
}

.config-label {
  min-width: 160px;
  font-weight: 600;
  font-size: 0.875rem;
  color: #475569;
  padding-top: 0.375rem;
}

.slider-wrapper {
  flex: 1;
}

.budget-slider {
  width: 100%;
  height: 6px;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
  -webkit-appearance: none;
  appearance: none;
  accent-color: #2563eb;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 0.5rem;
}

.budget-value {
  font-weight: 700;
  color: #2563eb;
  font-size: 0.875rem;
}

.lead-time-row {
  align-items: center;
}

.lead-time-input {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.number-input {
  width: 80px;
  padding: 0.375rem 0.625rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.938rem;
  color: #0f172a;
  outline: none;
  text-align: center;
}

.number-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.days-suffix {
  font-size: 0.875rem;
  color: #64748b;
}

/* Budget bar */
.budget-bar-container {
  margin-bottom: 1rem;
}

.budget-bar-label {
  font-size: 0.813rem;
  color: #64748b;
  margin-bottom: 0.375rem;
}

.budget-bar-label strong {
  color: #0f172a;
}

.budget-bar-track {
  height: 8px;
  background: #f1f5f9;
  border-radius: 4px;
  overflow: hidden;
}

.budget-bar-fill {
  height: 100%;
  border-radius: 4px;
  transition: width 0.3s ease, background 0.3s ease;
}

/* Budget summary in card header */
.budget-summary {
  font-size: 0.875rem;
}

.budget-status {
  color: #2563eb;
  font-weight: 600;
}

.budget-status.over-budget {
  color: #dc2626;
}

/* Table */
.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-check {
  width: 36px;
}

.col-sku {
  width: 120px;
}

.col-name {
  width: auto;
}

.col-trend {
  width: 100px;
}

.col-num {
  width: 100px;
  text-align: right;
}

.col-cost {
  width: 110px;
  text-align: right;
}

.restock-row {
  cursor: pointer;
}

.row-selected {
  background: #eff6ff !important;
}

.row-checkbox {
  cursor: pointer;
  accent-color: #2563eb;
  width: 15px;
  height: 15px;
}

.gap-cell {
  color: #dc2626;
  font-weight: 600;
}

code {
  font-family: monospace;
  font-size: 0.813rem;
  color: #475569;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
}

/* Actions row */
.actions-row {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
}

.selection-info {
  font-size: 0.875rem;
  color: #64748b;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.25rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}
</style>
