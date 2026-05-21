<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Review forecasted demand and plan restocking orders within your budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <div v-if="orderPlaced" class="success-banner">
        Order placed successfully! View it in the Orders tab.
      </div>

      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget Planner</h3>
        </div>
        <div class="budget-controls">
          <div class="budget-row">
            <label class="budget-label">Budget</label>
            <span class="budget-value">${{ formatCurrency(budget) }}</span>
          </div>
          <input
            type="range"
            class="budget-slider"
            min="0"
            :max="maxBudget"
            step="500"
            v-model.number="budget"
          />
          <div class="budget-meta">
            <span class="budget-meta-item">
              <span class="meta-label">Max budget:</span>
              <span class="meta-val">${{ formatCurrency(maxBudget) }}</span>
            </span>
            <span class="budget-meta-item">
              <span class="meta-label">Recommended cost:</span>
              <span class="meta-val">${{ formatCurrency(totalRecommendedCost) }}</span>
            </span>
            <span class="budget-meta-item" :class="{ negative: remainingBudget < 0 }">
              <span class="meta-label">Remaining:</span>
              <span class="meta-val">${{ formatCurrency(remainingBudget) }}</span>
            </span>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Restocking Items ({{ items.length }})</h3>
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
        <div class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th>Qty to Order</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in sortedItems" :key="item.item_sku">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ capitalize(item.trend) }}</span>
                </td>
                <td>{{ item.restock_quantity }}</td>
                <td>${{ item.unit_cost.toLocaleString() }}</td>
                <td>${{ item.item_total.toLocaleString() }}</td>
                <td>
                  <span
                    :class="['badge', isIncluded(item) ? 'success' : 'out-of-budget']">
                    {{ isIncluded(item) ? 'Included' : 'Out of budget' }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
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
    const loading = ref(true)
    const error = ref(null)
    const items = ref([])
    const budget = ref(0)
    const submitting = ref(false)
    const orderPlaced = ref(false)

    const maxBudget = computed(() => {
      const total = items.value.reduce((sum, item) => sum + item.item_total, 0)
      return Math.round(total / 500) * 500
    })

    const remainingBudget = computed(() => {
      return budget.value - totalRecommendedCost.value
    })

    const trendPriority = (trend) => {
      if (trend === 'increasing') return 0
      if (trend === 'stable') return 1
      return 2
    }

    const recommendedItems = computed(() => {
      const sorted = [...items.value].sort(
        (a, b) => trendPriority(a.trend) - trendPriority(b.trend)
      )
      const included = []
      let remaining = budget.value
      for (const item of sorted) {
        if (item.item_total <= remaining) {
          included.push(item)
          remaining -= item.item_total
        }
      }
      return included
    })

    const includedSkus = computed(() => {
      return new Set(recommendedItems.value.map(i => i.item_sku))
    })

    const sortedItems = computed(() => {
      return [...items.value].sort(
        (a, b) => trendPriority(a.trend) - trendPriority(b.trend)
      )
    })

    const totalRecommendedCost = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.item_total, 0)
    })

    const isIncluded = (item) => {
      return includedSkus.value.has(item.item_sku)
    }

    const formatCurrency = (value) => {
      return Math.abs(value).toLocaleString()
    }

    const capitalize = (str) => {
      if (!str) return str
      return str.charAt(0).toUpperCase() + str.slice(1)
    }

    const loadItems = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingItems()
        items.value = data.map(item => ({
          ...item,
          restock_quantity: item.forecasted_demand,
          item_total: item.unit_cost * item.forecasted_demand
        }))
        // Set initial budget to half of max after data loads
        const total = items.value.reduce((sum, item) => sum + item.item_total, 0)
        const max = Math.round(total / 500) * 500
        budget.value = Math.round(max / 2 / 500) * 500
      } catch (err) {
        error.value = 'Failed to load restocking items: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value) return
      submitting.value = true
      try {
        await api.createRestockingOrder({
          items: recommendedItems.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.restock_quantity,
            unit_cost: item.unit_cost,
            total_cost: item.item_total
          })),
          total_cost: totalRecommendedCost.value
        })
        orderPlaced.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadItems)

    return {
      loading,
      error,
      items,
      budget,
      maxBudget,
      remainingBudget,
      submitting,
      orderPlaced,
      recommendedItems,
      sortedItems,
      totalRecommendedCost,
      isIncluded,
      formatCurrency,
      capitalize,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-controls {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
}

.budget-meta {
  display: flex;
  gap: 2rem;
  flex-wrap: wrap;
}

.budget-meta-item {
  display: flex;
  gap: 0.375rem;
  font-size: 0.875rem;
}

.meta-label {
  color: #64748b;
}

.meta-val {
  font-weight: 600;
  color: #0f172a;
}

.budget-meta-item.negative .meta-val {
  color: #dc2626;
}

.budget-meta-item.negative::before {
  content: '-';
}

.restocking-table {
  table-layout: auto;
  width: 100%;
}

.place-order-btn {
  padding: 0.5rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.badge.out-of-budget {
  background: #f1f5f9;
  color: #64748b;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.938rem;
  margin-bottom: 1.25rem;
}
</style>
