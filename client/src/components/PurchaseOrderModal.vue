<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">{{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}</h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Backlog item context -->
            <div class="item-context">
              <div class="context-info">
                <h4 class="item-name">{{ translateProductName(backlogItem.item_name) }}</h4>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
              <div class="shortage-badge">
                <span class="shortage-label">Shortage</span>
                <span class="shortage-value">{{ shortage }} units</span>
              </div>
            </div>

            <!-- Create mode: form -->
            <form v-if="mode === 'create'" @submit.prevent="submitForm" class="po-form">
              <div class="form-group">
                <label class="form-label" for="supplier_name">Supplier Name</label>
                <input
                  id="supplier_name"
                  v-model="form.supplier_name"
                  type="text"
                  class="form-input"
                  placeholder="Enter supplier name"
                  required
                />
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="quantity">Quantity</label>
                  <input
                    id="quantity"
                    v-model.number="form.quantity"
                    type="number"
                    class="form-input"
                    min="1"
                    required
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="unit_cost">Unit Cost ($)</label>
                  <input
                    id="unit_cost"
                    v-model.number="form.unit_cost"
                    type="number"
                    class="form-input"
                    min="0"
                    step="0.01"
                    placeholder="0.00"
                    required
                  />
                </div>
              </div>

              <div class="form-group">
                <label class="form-label" for="expected_delivery_date">Expected Delivery Date</label>
                <input
                  id="expected_delivery_date"
                  v-model="form.expected_delivery_date"
                  type="date"
                  class="form-input"
                  required
                />
              </div>

              <div class="form-group">
                <label class="form-label" for="notes">Notes <span class="optional">(optional)</span></label>
                <textarea
                  id="notes"
                  v-model="form.notes"
                  class="form-input form-textarea"
                  placeholder="Additional notes..."
                  rows="3"
                />
              </div>

              <div v-if="submitError" class="error-message">{{ submitError }}</div>
            </form>

            <!-- View mode: read-only fields -->
            <div v-else-if="mode === 'view'" class="po-details">
              <div v-if="loadingPO" class="loading-state">Loading purchase order...</div>
              <div v-else-if="loadError" class="error-message">{{ loadError }}</div>
              <div v-else-if="existingPO" class="info-grid">
                <div class="info-item">
                  <div class="info-label">Supplier Name</div>
                  <div class="info-value">{{ existingPO.supplier_name }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Quantity</div>
                  <div class="info-value">{{ existingPO.quantity }} units</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Unit Cost</div>
                  <div class="info-value">${{ existingPO.unit_cost?.toFixed(2) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Expected Delivery</div>
                  <div class="info-value">{{ formatDate(existingPO.expected_delivery_date) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="status-badge" :class="existingPO.status">{{ existingPO.status }}</span>
                  </div>
                </div>

                <div class="info-item">
                  <div class="info-label">Created Date</div>
                  <div class="info-value">{{ formatDate(existingPO.created_date) }}</div>
                </div>

                <div v-if="existingPO.notes" class="info-item info-item-full">
                  <div class="info-label">Notes</div>
                  <div class="info-value">{{ existingPO.notes }}</div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">Close</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              :disabled="submitting"
              @click="submitForm"
            >
              {{ submitting ? 'Creating...' : 'Create Purchase Order' }}
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const { translateProductName } = useI18n()

const props = defineProps({
  isOpen: {
    type: Boolean,
    default: false
  },
  backlogItem: {
    type: Object,
    default: null
  },
  mode: {
    type: String,
    default: 'create'
  }
})

const emit = defineEmits(['close', 'po-created'])

const form = ref({
  supplier_name: '',
  quantity: 0,
  unit_cost: null,
  expected_delivery_date: '',
  notes: ''
})

const submitting = ref(false)
const submitError = ref(null)
const loadingPO = ref(false)
const loadError = ref(null)
const existingPO = ref(null)

const shortage = computed(() => {
  if (!props.backlogItem) return 0
  return props.backlogItem.quantity_needed - props.backlogItem.quantity_available
})

const resetForm = () => {
  form.value = {
    supplier_name: '',
    quantity: shortage.value > 0 ? shortage.value : 0,
    unit_cost: null,
    expected_delivery_date: '',
    notes: ''
  }
  submitError.value = null
}

const loadExistingPO = async () => {
  if (!props.backlogItem) return
  loadingPO.value = true
  loadError.value = null
  existingPO.value = null
  try {
    existingPO.value = await api.getPurchaseOrderByBacklogItem(props.backlogItem.id)
  } catch (err) {
    loadError.value = 'Failed to load purchase order details.'
    console.error(err)
  } finally {
    loadingPO.value = false
  }
}

watch(
  () => props.isOpen,
  (open) => {
    if (open) {
      if (props.mode === 'create') {
        resetForm()
      } else if (props.mode === 'view') {
        loadExistingPO()
      }
    }
  }
)

const close = () => {
  emit('close')
}

const submitForm = async () => {
  if (submitting.value) return
  submitting.value = true
  submitError.value = null
  try {
    const payload = {
      backlog_item_id: props.backlogItem.id,
      supplier_name: form.value.supplier_name,
      quantity: form.value.quantity,
      unit_cost: form.value.unit_cost,
      expected_delivery_date: form.value.expected_delivery_date,
      notes: form.value.notes || ''
    }
    const result = await api.createPurchaseOrder(payload)
    emit('po-created', result)
    emit('close')
  } catch (err) {
    submitError.value = err?.response?.data?.detail || 'Failed to create purchase order. Please try again.'
    console.error(err)
  } finally {
    submitting.value = false
  }
}

const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  const date = new Date(dateString)
  if (isNaN(date.getTime())) return 'N/A'
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 600px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 2rem;
}

/* Item context banner */
.item-context {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding: 1.25rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  margin-bottom: 1.75rem;
}

.context-info {
  min-width: 0;
}

.item-name {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.25rem 0;
}

.item-sku {
  font-size: 0.813rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.shortage-badge {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  flex-shrink: 0;
}

.shortage-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.shortage-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #dc2626;
}

/* Form styles */
.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.optional {
  font-weight: 400;
  text-transform: none;
  letter-spacing: 0;
  color: #94a3b8;
}

.form-input {
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 0.625rem 0.75rem;
  width: 100%;
  font-family: inherit;
  font-size: 0.938rem;
  color: #0f172a;
  background: white;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  box-sizing: border-box;
}

.form-input:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.form-textarea {
  resize: vertical;
  min-height: 80px;
}

/* View mode */
.loading-state {
  color: #64748b;
  font-size: 0.938rem;
  text-align: center;
  padding: 2rem 0;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.info-item-full {
  grid-column: 1 / -1;
}

.info-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.status-badge {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: capitalize;
}

.status-badge.pending {
  background: #fef3c7;
  color: #92400e;
}

.status-badge.approved {
  background: #dbeafe;
  color: #1e40af;
}

.status-badge.fulfilled {
  background: #dcfce7;
  color: #166534;
}

.status-badge.cancelled {
  background: #f1f5f9;
  color: #475569;
}

/* Error */
.error-message {
  padding: 0.75rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  color: #dc2626;
  font-size: 0.875rem;
}

/* Footer */
.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  border: none;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Modal transition animations */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
