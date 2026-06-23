<script setup>
import { ref, computed, watch } from 'vue'
import { onMounted } from 'vue'
import { api } from '../api'

const forecasts = ref([])
const inventoryList = ref([])
const loading = ref(false)
const error = ref(null)

const budget = ref(0)

const placing = ref(false)
const orderSuccess = ref(false)
const lastOrderNumber = ref(null)

const joinedItems = computed(() => {
  const inventoryMap = new Map(inventoryList.value.map(inv => [inv.sku, inv]))
  return forecasts.value
    .filter(f => inventoryMap.has(f.item_sku))
    .map(f => {
      const inv = inventoryMap.get(f.item_sku)
      return {
        sku: f.item_sku,
        name: f.item_name,
        forecasted_demand: f.forecasted_demand,
        unit_cost: inv.unit_cost,
        item_cost: f.forecasted_demand * inv.unit_cost,
        trend: f.trend
      }
    })
    .sort((a, b) => b.forecasted_demand - a.forecasted_demand)
})

const maxBudget = computed(() => {
  const total = joinedItems.value.reduce((sum, item) => sum + item.item_cost, 0)
  return Math.ceil(total / 1000) * 1000
})

watch(joinedItems, () => {
  if (budget.value === 0 && maxBudget.value > 0) {
    budget.value = Math.round(maxBudget.value * 0.5 / 1000) * 1000
  }
})

const selectedItems = computed(() => {
  let running = 0
  const result = []
  for (const item of joinedItems.value) {
    if (running + item.item_cost > budget.value) break
    running += item.item_cost
    result.push(item)
  }
  return result
})

const totalCost = computed(() =>
  selectedItems.value.reduce((sum, item) => sum + item.item_cost, 0)
)

const utilisationPct = computed(() =>
  Math.min(100, budget.value > 0 ? (totalCost.value / budget.value) * 100 : 0)
)

function formatCurrency(val) {
  return val.toLocaleString('en-US', { minimumFractionDigits: 0, maximumFractionDigits: 0 })
}

function isSelected(item) {
  return selectedItems.value.some(s => s.sku === item.sku)
}

async function placeOrder() {
  placing.value = true
  try {
    const response = await api.createRestockingOrder(
      selectedItems.value.map(i => ({
        sku: i.sku,
        name: i.name,
        quantity: i.forecasted_demand,
        unit_price: i.unit_cost
      }))
    )
    lastOrderNumber.value = response.order_number
    orderSuccess.value = true
  } catch (err) {
    console.error('Failed to place restocking order:', err)
  } finally {
    placing.value = false
  }
}

const loadData = async () => {
  loading.value = true
  error.value = null
  try {
    const [forecastData, inventoryData] = await Promise.all([
      api.getDemandForecasts(),
      api.getInventory({})
    ])
    forecasts.value = forecastData
    inventoryList.value = inventoryData
  } catch (err) {
    error.value = 'Failed to load restocking data'
    console.error(err)
  } finally {
    loading.value = false
  }
}

onMounted(() => loadData())
</script>

<template>
  <div class="restocking-view">
    <div class="page-header">
      <h1>Restocking Planner</h1>
      <p class="subtitle">Set your available budget and we'll recommend which items to restock based on forecasted demand.</p>
    </div>

    <div v-if="error" class="error">{{ error }}</div>

    <div class="budget-card">
      <div class="budget-header">
        <span class="budget-label">Available Budget</span>
        <span class="budget-value">${{ formatCurrency(budget) }}</span>
      </div>
      <input
        type="range"
        v-model.number="budget"
        :min="0"
        :max="maxBudget"
        :step="1000"
        class="budget-slider"
      />
      <div class="budget-meta">
        <span>$0</span>
        <span>${{ formatCurrency(maxBudget) }} (all items)</span>
      </div>
      <div class="utilisation-bar">
        <div class="utilisation-fill" :style="{ width: utilisationPct + '%' }"></div>
      </div>
      <div class="utilisation-label">
        <span>{{ selectedItems.length }} item{{ selectedItems.length !== 1 ? 's' : '' }} selected</span>
        <span>${{ formatCurrency(totalCost) }} / ${{ formatCurrency(budget) }} used</span>
      </div>
    </div>

    <div class="items-card">
      <h2>Recommended Restocking Items</h2>
      <div v-if="loading" class="loading">Loading forecast data...</div>
      <table v-else class="items-table">
        <thead>
          <tr>
            <th>Item</th>
            <th>SKU</th>
            <th>Trend</th>
            <th class="num">Forecast Qty</th>
            <th class="num">Unit Cost</th>
            <th class="num">Line Cost</th>
            <th>Status</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="item in joinedItems"
            :key="item.sku"
            :class="{ included: isSelected(item), excluded: !isSelected(item) }"
          >
            <td class="item-name">{{ item.name }}</td>
            <td class="sku">{{ item.sku }}</td>
            <td><span :class="['trend-badge', 'trend-' + item.trend]">{{ item.trend }}</span></td>
            <td class="num">{{ item.forecasted_demand }}</td>
            <td class="num">${{ item.unit_cost.toFixed(2) }}</td>
            <td class="num">${{ formatCurrency(item.item_cost) }}</td>
            <td>
              <span v-if="isSelected(item)" class="status-in">In order</span>
              <span v-else class="status-out">Over budget</span>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <div class="order-footer">
      <div v-if="orderSuccess" class="success-banner">
        Order {{ lastOrderNumber }} placed successfully. Check the Orders tab to track delivery.
      </div>
      <button
        class="place-order-btn"
        :disabled="selectedItems.length === 0 || orderSuccess || placing"
        @click="placeOrder"
      >
        {{ placing ? 'Placing order...' : 'Place Order' }}
      </button>
    </div>
  </div>
</template>

<style scoped>
.restocking-view {
  padding: 0;
}

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h1 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 0.375rem;
}

.page-header .subtitle {
  color: #64748b;
  font-size: 0.938rem;
  border: none;
  padding: 0;
}

/* Budget card */
.budget-card {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 1.5rem;
  margin-bottom: 1.25rem;
}

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 1rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.budget-value {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  margin-bottom: 0.5rem;
  appearance: none;
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #38bdf8;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  transition: background 0.15s;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #0ea5e9;
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #38bdf8;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-meta {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
  margin-bottom: 1rem;
}

.utilisation-bar {
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.utilisation-fill {
  height: 100%;
  background: #3b82f6;
  border-radius: 3px;
  transition: width 0.2s ease;
}

.utilisation-label {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
}

/* Items card */
.items-card {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 1.5rem;
  margin-bottom: 1.25rem;
}

.items-card h2 {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
}

.items-table {
  width: 100%;
  border-collapse: collapse;
}

.items-table th {
  text-align: left;
  padding: 0.5rem 1rem;
  font-size: 0.688rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

.items-table th.num,
.items-table td.num {
  text-align: right;
}

.items-table td {
  padding: 12px 16px;
  border-bottom: 1px solid #f1f5f9;
  font-size: 0.875rem;
  color: #334155;
}

.items-table tbody tr {
  transition: opacity 0.15s ease, background-color 0.15s ease;
}

.items-table tbody tr.included {
  opacity: 1;
}

.items-table tbody tr.excluded {
  opacity: 0.45;
}

.items-table tbody tr:hover {
  background: #f8fafc;
}

.item-name {
  font-weight: 500;
  color: #0f172a;
}

.sku {
  font-family: monospace;
  font-size: 0.813rem;
  color: #64748b;
}

/* Trend badge */
.trend-badge {
  display: inline-block;
  padding: 0.25rem 0.625rem;
  border-radius: 999px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: capitalize;
}

.trend-increasing {
  background: #d1fae5;
  color: #065f46;
}

.trend-stable {
  background: #dbeafe;
  color: #1e40af;
}

.trend-decreasing {
  background: #fee2e2;
  color: #991b1b;
}

/* Status */
.status-in {
  font-size: 0.813rem;
  font-weight: 600;
  color: #059669;
}

.status-out {
  font-size: 0.813rem;
  color: #94a3b8;
}

/* Order footer */
.order-footer {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.75rem;
}

.success-banner {
  width: 100%;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 12px 16px;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
}

.place-order-btn {
  padding: 12px 32px;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
  opacity: 0.7;
}
</style>
