<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your available budget and select order quantities for increasing-demand items</p>
    </div>

    <!-- Budget Card -->
    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
        <span class="budget-display">${{ budget.toLocaleString() }}</span>
      </div>
      <div class="slider-wrap">
        <span class="slider-bound">$0</span>
        <input
          type="range"
          v-model.number="budget"
          min="0"
          max="500000"
          step="1000"
          class="budget-slider"
        />
        <span class="slider-bound">$500,000</span>
      </div>
      <div class="budget-progress-track">
        <div
          class="budget-progress-fill"
          :class="{ over: committed > budget }"
          :style="{ width: Math.min((committed / Math.max(budget, 1)) * 100, 100) + '%' }"
        ></div>
      </div>
      <div class="budget-summary">
        <div class="bstat">
          <span class="bstat-label">Budget</span>
          <span class="bstat-value">${{ budget.toLocaleString() }}</span>
        </div>
        <div class="bstat">
          <span class="bstat-label">Committed</span>
          <span class="bstat-value" :class="{ over: committed > budget }">
            ${{ Math.round(committed).toLocaleString() }}
          </span>
        </div>
        <div class="bstat">
          <span class="bstat-label">Remaining</span>
          <span class="bstat-value" :class="remaining < 0 ? 'over' : 'under'">
            {{ remaining < 0 ? '-' : '' }}${{ Math.abs(Math.round(remaining)).toLocaleString() }}
          </span>
        </div>
      </div>
    </div>

    <!-- Items Table -->
    <div v-if="loading" class="loading">Loading demand forecasts...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            Increasing Demand Items
            <span class="count-chip">{{ forecastItems.length }}</span>
          </h3>
          <span class="card-meta">Items with forecasted demand higher than current · Period: Next 30 days</span>
        </div>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-sku">SKU</th>
                <th class="col-name">Item</th>
                <th class="col-demand">Current</th>
                <th class="col-demand">Forecasted</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-qty">Quantity</th>
                <th class="col-sub">Subtotal</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in forecastItems"
                :key="item.item_sku"
                :class="{ 'row-active': (quantities[item.item_sku] || 0) > 0 }"
              >
                <td class="col-sku"><span class="sku-tag">{{ item.item_sku }}</span></td>
                <td class="col-name">{{ item.item_name }}</td>
                <td class="col-demand">{{ item.current_demand.toLocaleString() }}</td>
                <td class="col-demand">
                  {{ item.forecasted_demand.toLocaleString() }}
                  <span class="trend-arrow">↑</span>
                </td>
                <td class="col-cost">
                  <span v-if="item.unit_cost">${{ item.unit_cost.toFixed(2) }}</span>
                  <span v-else class="no-data">—</span>
                </td>
                <td class="col-qty">
                  <input
                    type="number"
                    v-model.number="quantities[item.item_sku]"
                    min="0"
                    step="1"
                    class="qty-input"
                    placeholder="0"
                  />
                </td>
                <td class="col-sub">
                  <span v-if="(quantities[item.item_sku] || 0) > 0 && item.unit_cost" class="subtotal-val">
                    ${{ Math.round((quantities[item.item_sku] || 0) * item.unit_cost).toLocaleString() }}
                  </span>
                  <span v-else class="no-data">—</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="order-footer">
          <div class="order-total-area">
            <span class="order-total-label">Order Total</span>
            <span class="order-total-value" :class="{ over: committed > budget }">
              ${{ Math.round(committed).toLocaleString() }}
            </span>
            <span v-if="committed > budget" class="over-msg">
              Over budget by ${{ Math.round(committed - budget).toLocaleString() }}
            </span>
          </div>
          <div class="footer-actions">
            <button class="btn-reset" @click="resetQuantities" :disabled="committed === 0">
              Clear
            </button>
            <button
              class="btn-place-order"
              :disabled="!canPlaceOrder"
              @click="placeOrder"
            >
              <span v-if="submitting">Submitting...</span>
              <span v-else>Place Order</span>
            </button>
          </div>
        </div>
      </div>

      <div v-if="orderSuccess" class="success-banner">
        Restocking order submitted. View it in the Orders tab under "Restocking Orders".
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
    const forecastItems = ref([])
    const quantities = ref({})
    const budget = ref(100000)
    const submitting = ref(false)
    const orderSuccess = ref(false)

    const committed = computed(() =>
      forecastItems.value.reduce((sum, item) => {
        const qty = quantities.value[item.item_sku] || 0
        return sum + qty * (item.unit_cost || 0)
      }, 0)
    )

    const remaining = computed(() => budget.value - committed.value)

    const canPlaceOrder = computed(() => {
      const hasItems = forecastItems.value.some(
        item => (quantities.value[item.item_sku] || 0) > 0
      )
      return hasItems && committed.value <= budget.value && !submitting.value
    })

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null

        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])

        // SKU → unit_cost lookup; inventory may have multiple warehouses per SKU
        const inventoryMap = {}
        inventory.forEach(item => {
          if (!inventoryMap[item.sku]) inventoryMap[item.sku] = item.unit_cost
        })

        forecastItems.value = forecasts
          .filter(f => f.trend === 'increasing')
          .map(f => ({ ...f, unit_cost: inventoryMap[f.item_sku] ?? null }))

        const initial = {}
        forecastItems.value.forEach(item => { initial[item.item_sku] = 0 })
        quantities.value = initial
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const resetQuantities = () => {
      const reset = {}
      forecastItems.value.forEach(item => { reset[item.item_sku] = 0 })
      quantities.value = reset
    }

    const placeOrder = async () => {
      if (!canPlaceOrder.value) return
      submitting.value = true
      try {
        const items = forecastItems.value
          .filter(item => (quantities.value[item.item_sku] || 0) > 0)
          .map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: quantities.value[item.item_sku],
            unit_cost: item.unit_cost || 0
          }))

        await api.submitRestockingOrder({
          items,
          total_value: Math.round(committed.value * 100) / 100,
          budget: budget.value
        })

        orderSuccess.value = true
        resetQuantities()
        setTimeout(() => { orderSuccess.value = false }, 6000)
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      loading, error, forecastItems, quantities,
      budget, committed, remaining, canPlaceOrder,
      submitting, orderSuccess, resetQuantities, placeOrder
    }
  }
}
</script>

<style scoped>
/* ── Budget Card ── */
.budget-card { margin-bottom: 1.25rem; }

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.slider-wrap {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin: 1rem 0 0.75rem;
}

.slider-bound {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-progress-track {
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
  overflow: hidden;
  margin-bottom: 1rem;
}

.budget-progress-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 3px;
  transition: width 0.2s ease, background 0.2s ease;
}

.budget-progress-fill.over { background: #dc2626; }

.budget-summary {
  display: flex;
  gap: 2.5rem;
}

.bstat { display: flex; flex-direction: column; gap: 2px; }

.bstat-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.bstat-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.bstat-value.over  { color: #dc2626; }
.bstat-value.under { color: #059669; }

/* ── Table ── */
.card-meta {
  font-size: 0.813rem;
  color: #64748b;
}

.count-chip {
  display: inline-block;
  background: #e0e7ff;
  color: #3730a3;
  font-size: 0.75rem;
  font-weight: 600;
  padding: 1px 8px;
  border-radius: 10px;
  margin-left: 6px;
  vertical-align: middle;
}

.restock-table { table-layout: fixed; width: 100%; }

.col-sku    { width: 110px; }
.col-name   { width: auto; }
.col-demand { width: 110px; }
.col-cost   { width: 100px; }
.col-qty    { width: 100px; }
.col-sub    { width: 110px; }

.sku-tag {
  font-family: 'SF Mono', 'Fira Code', monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  padding: 2px 6px;
  color: #475569;
}

.trend-arrow {
  color: #059669;
  font-weight: 700;
  margin-left: 4px;
}

.no-data { color: #94a3b8; }

.qty-input {
  width: 72px;
  padding: 4px 8px;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  text-align: right;
  transition: border-color 0.15s;
}

.qty-input:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 2px rgba(37, 99, 235, 0.12);
}

.subtotal-val { font-weight: 600; color: #0f172a; }

.row-active td { background: #f0f9ff; }

/* ── Footer ── */
.order-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 0.75rem 0.25rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
}

.order-total-area {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
}

.order-total-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 600;
}

.order-total-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.order-total-value.over { color: #dc2626; }

.over-msg {
  font-size: 0.813rem;
  color: #dc2626;
  font-weight: 500;
}

.footer-actions { display: flex; gap: 0.75rem; }

.btn-reset {
  padding: 0.5rem 1rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  background: white;
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
}

.btn-reset:hover:not(:disabled) {
  border-color: #cbd5e1;
  background: #f8fafc;
  color: #0f172a;
}

.btn-reset:disabled { opacity: 0.4; cursor: default; }

.btn-place-order {
  padding: 0.5rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-place-order:hover:not(:disabled) { background: #1d4ed8; }

.btn-place-order:disabled {
  background: #94a3b8;
  cursor: default;
}

/* ── Success banner ── */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
  margin-top: 1rem;
}
</style>
