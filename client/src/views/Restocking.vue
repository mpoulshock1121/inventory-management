<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget card -->
      <div class="card">
        <div class="budget-card-header">
          <span class="card-title">{{ t('restocking.budget') }}</span>
          <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
        <input
          type="range"
          min="0"
          max="50000"
          step="1000"
          v-model.number="budget"
        />
        <div class="budget-labels">
          <span>{{ currencySymbol }}0</span>
          <span>{{ currencySymbol }}50,000</span>
        </div>
      </div>

      <!-- Stats row -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.totalCost') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ totalCost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ remainingBudget.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.itemsSelected') }}</div>
          <div class="stat-value">{{ recommendedItems.length }}</div>
        </div>
      </div>

      <!-- Success banner -->
      <div v-if="orderSuccess" class="success-banner">
        <span>Order {{ submittedOrderNumber }} submitted successfully. Check the Orders tab for delivery status.</span>
        <button class="btn-reset" @click="resetOrder">Place Another Order</button>
      </div>

      <!-- Recommended items card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendedItems') }}</h3>
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.submitting') : t('restocking.placeOrder') }}
          </button>
        </div>
        <div v-if="recommendedItems.length === 0" class="loading">
          {{ t('restocking.noItems') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.sku') }}</th>
                <th>{{ t('restocking.itemName') }}</th>
                <th>{{ t('restocking.forecastedDemand') }}</th>
                <th>{{ t('restocking.reorderQty') }}</th>
                <th>{{ t('restocking.unitCost') }}</th>
                <th>{{ t('restocking.lineCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.forecasted_demand }}</td>
                <td>{{ item.reorder_qty }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                <td>{{ currencySymbol }}{{ item.line_cost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
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
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    // refs
    const budget = ref(10000)
    const forecasts = ref([])
    const inventoryItems = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const orderSuccess = ref(false)
    const submittedOrderNumber = ref('')

    const { t, currentCurrency } = useI18n()
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    // Join demand forecasts with inventory by SKU
    const inventoryBySku = computed(() => {
      const map = {}
      for (const item of inventoryItems.value) {
        map[item.sku] = item
      }
      return map
    })

    // Enrich forecasts with inventory cost/qty data, sort by forecasted_demand desc
    const enrichedForecasts = computed(() => {
      return forecasts.value
        .map(f => {
          const inv = inventoryBySku.value[f.item_sku]
          if (!inv) return null
          return {
            ...f,
            reorder_qty: inv.reorder_point,
            unit_cost: inv.unit_cost,
            line_cost: inv.reorder_point * inv.unit_cost,
            category: inv.category,
            warehouse: inv.warehouse
          }
        })
        .filter(Boolean)
        .sort((a, b) => b.forecasted_demand - a.forecasted_demand)
    })

    // Greedy fill: iterate sorted items, include if cost fits in remaining budget
    // Does NOT stop on first miss — cheaper items later may still fit
    const recommendedItems = computed(() => {
      let remaining = budget.value
      return enrichedForecasts.value.filter(item => {
        if (item.line_cost <= remaining) {
          remaining -= item.line_cost
          return true
        }
        return false
      })
    })

    const totalCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.line_cost, 0)
    )
    const remainingBudget = computed(() => budget.value - totalCost.value)

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()  // no filters — need all items for SKU join
        ])
        forecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const resetOrder = () => {
      orderSuccess.value = false
      submittedOrderNumber.value = ''
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value) return
      submitting.value = true
      error.value = null
      try {
        const now = new Date()
        const delivery = new Date(now)
        delivery.setDate(delivery.getDate() + 14)
        const orderNumber = `RST-${now.getFullYear()}-${String(Math.floor(Math.random() * 9000) + 1000)}`

        await api.createOrder({
          order_number: orderNumber,
          customer: 'Internal Restocking',
          items: recommendedItems.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.reorder_qty,
            unit_price: item.unit_cost
          })),
          status: 'Restocking',
          order_date: now.toISOString(),
          expected_delivery: delivery.toISOString(),
          total_value: totalCost.value,
          actual_delivery: null,
          warehouse: 'all',
          category: 'all'
        })

        submittedOrderNumber.value = orderNumber
        orderSuccess.value = true
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      budget,
      loading,
      error,
      submitting,
      orderSuccess,
      submittedOrderNumber,
      recommendedItems,
      totalCost,
      remainingBudget,
      currencySymbol,
      placeOrder,
      resetOrder
    }
  }
}
</script>

<style scoped>
.budget-card-header {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.25rem;
}

.budget-value {
  font-size: 2rem;
  font-weight: 700;
  color: #2563eb;
}

input[type='range'] {
  width: 100%;
  accent-color: #3b82f6;
  height: 6px;
  cursor: pointer;
  margin: 1rem 0;
}

.budget-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
}

.success-banner {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1.25rem;
  font-weight: 500;
  font-size: 0.938rem;
  display: flex;
  align-items: center;
}

.btn-reset {
  background: transparent;
  border: 1px solid #065f46;
  color: #065f46;
  padding: 0.375rem 0.875rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  cursor: pointer;
  margin-left: 1rem;
}
.btn-reset:hover {
  background: #d1fae5;
}

.place-order-btn {
  background: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.25rem;
  font-weight: 600;
  font-size: 0.938rem;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
