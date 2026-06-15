<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const { currentCurrency } = useI18n()
const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

const loading = ref(true)
const error = ref(null)
const submitting = ref(false)
const successBanner = ref(null)

const recommendations = ref([])
const budget = ref(100000)

// Track manual checkbox overrides: sku -> true (force-include) | false (force-exclude)
const manualOverrides = ref(new Map())

// Greedy auto-selection within budget (API already returns high → medium → low by total_cost asc)
const autoSelected = computed(() => {
  const selected = new Set()
  let remaining = budget.value
  for (const item of recommendations.value) {
    if (item.total_cost <= remaining) {
      selected.add(item.sku)
      remaining -= item.total_cost
    }
  }
  return selected
})

// Reset manual overrides whenever the slider moves — slider is a deliberate "start fresh" signal
watch(budget, () => {
  manualOverrides.value = new Map()
})

function isSelected(item) {
  if (manualOverrides.value.has(item.sku)) return manualOverrides.value.get(item.sku)
  return autoSelected.value.has(item.sku)
}

function toggleItem(item) {
  const current = isSelected(item)
  const next = !current
  const overrides = new Map(manualOverrides.value)
  // If next matches what auto-selection would give, remove the override (keep it clean)
  if (next === autoSelected.value.has(item.sku)) {
    overrides.delete(item.sku)
  } else {
    overrides.set(item.sku, next)
  }
  manualOverrides.value = overrides
}

const selectedItems = computed(() => recommendations.value.filter(r => isSelected(r)))
const totalCost = computed(() => selectedItems.value.reduce((sum, r) => sum + r.total_cost, 0))
const budgetUsedPct = computed(() => budget.value > 0 ? Math.min(100, (totalCost.value / budget.value) * 100) : 0)
const overBudget = computed(() => totalCost.value > budget.value)

async function loadRecommendations() {
  loading.value = true
  error.value = null
  try {
    recommendations.value = await api.getRestockingRecommendations()
  } catch (err) {
    error.value = 'Failed to load recommendations: ' + err.message
  } finally {
    loading.value = false
  }
}

async function placeOrder() {
  if (selectedItems.value.length === 0 || submitting.value) return
  submitting.value = true
  successBanner.value = null
  try {
    const items = selectedItems.value.map(r => ({
      sku: r.sku,
      name: r.name,
      quantity: r.recommended_quantity,
      unit_cost: r.unit_cost,
      total_cost: r.total_cost
    }))
    const order = await api.submitRestockingOrder(items)
    const deliveryDate = new Date(order.expected_delivery).toLocaleDateString('en-US', {
      year: 'numeric', month: 'short', day: 'numeric'
    })
    successBanner.value = `Restocking order ${order.order_number} submitted. Expected delivery: ${deliveryDate}.`
    // Reset overrides after successful order
    manualOverrides.value = new Map()
  } catch (err) {
    error.value = 'Failed to place order: ' + err.message
  } finally {
    submitting.value = false
  }
}

function formatCurrency(value) {
  return currencySymbol.value + value.toLocaleString('en-US', { minimumFractionDigits: 0 })
}

function formatBudgetSlider(value) {
  return currencySymbol.value + Number(value).toLocaleString('en-US')
}

onMounted(loadRecommendations)
</script>

<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend items to restock based on demand forecasts and current stock levels.</p>
    </div>

    <!-- Success banner -->
    <div v-if="successBanner" class="success-banner">
      <span>{{ successBanner }}</span>
      <button class="dismiss-btn" @click="successBanner = null">Dismiss</button>
    </div>

    <!-- Budget card -->
    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
        <span class="budget-display">{{ formatBudgetSlider(budget) }}</span>
      </div>
      <input
        type="range"
        class="budget-slider"
        min="0"
        max="500000"
        step="10000"
        v-model.number="budget"
      />
      <div class="slider-labels">
        <span>{{ currencySymbol }}0</span>
        <span>{{ currencySymbol }}250,000</span>
        <span>{{ currencySymbol }}500,000</span>
      </div>
    </div>

    <!-- Loading / error -->
    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <template v-else>

      <!-- Budget summary bar -->
      <div class="card budget-summary">
        <div class="summary-row">
          <span class="summary-text">
            <strong>{{ selectedItems.length }}</strong> items selected
            &nbsp;·&nbsp;
            <strong :class="{ 'over-budget': overBudget }">{{ formatCurrency(totalCost) }}</strong>
            of {{ formatCurrency(budget) }} budget used
            &nbsp;·&nbsp;
            <span :class="overBudget ? 'over-budget' : 'remaining'">
              {{ overBudget ? formatCurrency(totalCost - budget) + ' over budget' : formatCurrency(budget - totalCost) + ' remaining' }}
            </span>
          </span>
          <button
            class="place-order-btn"
            :disabled="selectedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : `Place Order (${selectedItems.length} items · ${formatCurrency(totalCost)})` }}
          </button>
        </div>
        <div class="progress-bar-track">
          <div
            class="progress-bar-fill"
            :class="{ 'over': overBudget }"
            :style="{ width: budgetUsedPct + '%' }"
          ></div>
        </div>
      </div>

      <!-- Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ selectedItems.length }} selected)</h3>
        </div>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-item">Item</th>
                <th class="col-cat">Category</th>
                <th class="col-wh">Warehouse</th>
                <th class="col-num">Current Stock</th>
                <th class="col-num">Reorder Point</th>
                <th class="col-badge">Trend</th>
                <th class="col-badge">Urgency</th>
                <th class="col-num">Qty to Order</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-cost">Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ 'row-selected': isSelected(item), 'row-muted': !isSelected(item) }"
                @click="toggleItem(item)"
                class="clickable-row"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="isSelected(item)"
                    @click.stop="toggleItem(item)"
                  />
                </td>
                <td class="col-item">
                  <div class="item-name">{{ item.name }}</div>
                  <div class="item-sku">{{ item.sku }}</div>
                </td>
                <td class="col-cat">{{ item.category }}</td>
                <td class="col-wh">{{ item.warehouse }}</td>
                <td class="col-num">{{ item.current_stock.toLocaleString() }}</td>
                <td class="col-num">{{ item.reorder_point.toLocaleString() }}</td>
                <td class="col-badge">
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-badge">
                  <span :class="['badge', item.urgency]">{{ item.urgency }}</span>
                </td>
                <td class="col-num"><strong>{{ item.recommended_quantity.toLocaleString() }}</strong></td>
                <td class="col-cost">{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-cost"><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </template>
  </div>
</template>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

/* Success banner */
.success-banner {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.9rem;
  font-weight: 500;
}

.dismiss-btn {
  background: none;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 6px;
  padding: 0.25rem 0.75rem;
  font-size: 0.8rem;
  cursor: pointer;
  font-weight: 500;
}
.dismiss-btn:hover { background: #a7f3d0; }

/* Budget card */
.budget-card .card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.02em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  margin: 0.75rem 0 0.5rem;
  accent-color: #2563eb;
  cursor: pointer;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}

/* Budget summary */
.budget-summary {
  margin-bottom: 1.25rem;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
  flex-wrap: wrap;
  gap: 0.75rem;
}

.summary-text {
  font-size: 0.875rem;
  color: #475569;
}

.over-budget { color: #dc2626; }
.remaining   { color: #059669; }

.progress-bar-track {
  height: 6px;
  background: #e2e8f0;
  border-radius: 99px;
  overflow: hidden;
}

.progress-bar-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 99px;
  transition: width 0.3s ease;
}

.progress-bar-fill.over { background: #dc2626; }

/* Place order button */
.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.625rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
  white-space: nowrap;
}
.place-order-btn:hover:not(:disabled) { background: #1d4ed8; }
.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

/* Table */
.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-check { width: 40px; }
.col-item  { width: 200px; }
.col-cat   { width: 130px; }
.col-wh    { width: 130px; }
.col-num   { width: 110px; text-align: right; }
.col-badge { width: 110px; }
.col-cost  { width: 110px; text-align: right; }

.item-name { font-weight: 500; color: #0f172a; font-size: 0.875rem; }
.item-sku  { font-size: 0.75rem; color: #94a3b8; margin-top: 2px; }

.clickable-row { cursor: pointer; }

.row-selected { background: #ffffff; }
.row-muted    { opacity: 0.45; }
.row-muted:hover { opacity: 0.65; }
</style>
