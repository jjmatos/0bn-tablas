<template>
  <div>
    <!-- Function toggles -->
    <div class="mb-6 space-y-3 bg-gray-800 p-4 rounded-lg">
      <h3 class="font-semibold text-gray-300 mb-2">Table Features</h3>
      <div class="flex flex-wrap gap-4">
        <label class="flex items-center space-x-2">
          <input
            type="checkbox"
            v-model="features.search"
            class="rounded border-gray-600 text-indigo-600 
                   focus:ring-indigo-500 focus:ring-offset-gray-800"
          />
          <span class="text-gray-300">Enable Search</span>
        </label>
        <label class="flex items-center space-x-2">
          <input
            type="checkbox"
            v-model="features.sort"
            class="rounded border-gray-600 text-indigo-600 
                   focus:ring-indigo-500 focus:ring-offset-gray-800"
          />
          <span class="text-gray-300">Enable Sorting</span>
        </label>
        <label class="flex items-center space-x-2">
          <input
            type="checkbox"
            v-model="features.edit"
            class="rounded border-gray-600 text-indigo-600 
                   focus:ring-indigo-500 focus:ring-offset-gray-800"
          />
          <span class="text-gray-300">Enable Cell Editing</span>
        </label>
      </div>
      <div class="text-sm text-gray-400">
        Shortcuts: Ctrl+1, Ctrl+2, Ctrl+3 to copy values from first row columns
      </div>
    </div>

    <!-- Search input -->
    <div v-if="features.search" class="mb-4">
      <input
        type="text"
        v-model="searchQuery"
        :placeholder="'Search in ' + (props.headers[1] || 'all columns') + '...'"
        class="w-full px-4 py-2 bg-gray-700 border border-gray-600 rounded-lg 
               text-white placeholder-gray-400 focus:outline-none focus:ring-2 
               focus:ring-indigo-500 focus:border-transparent"
      />
    </div>

    <div class="overflow-x-auto rounded-lg border border-gray-700">
      <table class="w-full border-collapse">
        <thead>
          <tr>
            <th class="bg-gray-700 px-4 py-3 border-b border-gray-600">
              <input
                type="checkbox"
                :checked="allSelected"
                @change="toggleAllRows"
                class="w-4 h-4 rounded border-gray-600 text-indigo-600 
                       focus:ring-indigo-500 focus:ring-offset-gray-700"
              />
            </th>
            <th v-for="(header, index) in props.headers" 
                :key="header"
                @click="features.sort && sort(header)"
                class="bg-gray-700 px-4 py-3 border-b border-gray-600 
                       text-left"
                :class="{ 'cursor-pointer hover:bg-gray-600 transition-colors': features.sort }">
              <div class="flex items-center justify-between">
                <span class="font-semibold">{{ header }}</span>
                <div class="flex items-center space-x-2">
                  <span class="text-xs text-gray-400">(Ctrl+{{ index + 1 }})</span>
                  <span v-if="features.sort && sortColumn === header" class="ml-2">
                    {{ sortDirection === 'asc' ? '↑' : '↓' }}
                  </span>
                </div>
              </div>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(row, rowIndex) in paginatedData" 
              :key="rowIndex"
              class="hover:bg-gray-700/50 transition-colors"
              :class="{ 'bg-indigo-900/30': selectedRows.includes(rowIndex + currentPage * pageSize) }">
            <td class="px-4 py-3 border-b border-gray-700">
              <input
                type="checkbox"
                :checked="selectedRows.includes(rowIndex + currentPage * pageSize)"
                @change="toggleRow(rowIndex + currentPage * pageSize)"
                class="w-4 h-4 rounded border-gray-600 text-indigo-600 
                       focus:ring-indigo-500 focus:ring-offset-gray-700"
              />
            </td>
            <td v-for="header in props.headers" 
                :key="header"
                class="px-4 py-3 border-b border-gray-700 relative group"
                @dblclick="features.edit && startEditing(rowIndex, header, row[header])">
              <div v-if="isEditing(rowIndex, header)" class="flex">
                <input
                  ref="editInput"
                  v-model="editValue"
                  class="w-full px-2 py-1 bg-gray-600 border border-gray-500 rounded 
                         text-white focus:outline-none focus:ring-2 focus:ring-indigo-500"
                  @keyup.enter="saveEdit(row)"
                  @keyup.esc="cancelEdit"
                  @blur="cancelEdit"
                  v-focus
                />
              </div>
              <div v-else class="flex items-center justify-between">
                <span>{{ row[header] }}</span>
                <button
                  @click="copyValue(row[header])"
                  class="opacity-0 group-hover:opacity-100 px-2 py-1 bg-indigo-600 
                         text-white text-xs rounded hover:bg-indigo-700 
                         transition-all ml-2"
                  :title="`Copy ${header}`"
                >
                  Copy
                </button>
              </div>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Pagination -->
    <div class="mt-4 flex items-center justify-between text-gray-400">
      <div>
        {{ selectedRows.length }} row(s) selected
      </div>
      <div class="flex items-center space-x-4">
        <div class="text-sm">
          Page {{ currentPage + 1 }} of {{ totalPages }}
        </div>
        <div class="flex space-x-2">
          <button
            @click="currentPage = 0"
            :disabled="currentPage === 0"
            class="px-3 py-1 rounded bg-gray-700 hover:bg-gray-600 disabled:opacity-50 
                   disabled:cursor-not-allowed transition-colors"
          >
            First
          </button>
          <button
            @click="currentPage--"
            :disabled="currentPage === 0"
            class="px-3 py-1 rounded bg-gray-700 hover:bg-gray-600 disabled:opacity-50 
                   disabled:cursor-not-allowed transition-colors"
          >
            Previous
          </button>
          <button
            @click="currentPage++"
            :disabled="currentPage >= totalPages - 1"
            class="px-3 py-1 rounded bg-gray-700 hover:bg-gray-600 disabled:opacity-50 
                   disabled:cursor-not-allowed transition-colors"
          >
            Next
          </button>
          <button
            @click="currentPage = totalPages - 1"
            :disabled="currentPage >= totalPages - 1"
            class="px-3 py-1 rounded bg-gray-700 hover:bg-gray-600 disabled:opacity-50 
                   disabled:cursor-not-allowed transition-colors"
          >
            Last
          </button>
        </div>
      </div>
    </div>

    <!-- Copy success notification -->
    <div
      v-if="showCopyNotification"
      class="fixed bottom-4 right-4 bg-green-500 text-white px-4 py-2 rounded-lg 
             shadow-lg transition-opacity duration-300"
      :class="{ 'opacity-0': !showCopyNotification }"
    >
      {{ copyMessage }}
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const props = defineProps({
  headers: {
    type: Array,
    required: true
  },
  data: {
    type: Array,
    required: true
  }
})

const emit = defineEmits(['selection-change', 'update:data'])

// Feature toggles
const features = ref({
  search: true,
  sort: true,
  edit: true
})

// Editing state
const editingCell = ref(null)
const editValue = ref('')
const editInput = ref(null)

// Copy notification state
const showCopyNotification = ref(false)
const copyMessage = ref('')
let notificationTimeout = null

// Copy functionality
const copyValue = async (value, columnName = '') => {
  try {
    await navigator.clipboard.writeText(String(value))
    copyMessage.value = columnName 
      ? `Copied ${columnName}: ${value}`
      : 'Copied to clipboard!'
    showCopySuccess()
  } catch (err) {
    console.error('Failed to copy text: ', err)
  }
}

const showCopySuccess = () => {
  showCopyNotification.value = true
  if (notificationTimeout) {
    clearTimeout(notificationTimeout)
  }
  notificationTimeout = setTimeout(() => {
    showCopyNotification.value = false
  }, 2000)
}

// Keyboard shortcut handler
const handleKeyboardShortcut = (e) => {
  // Handle Ctrl+1, Ctrl+2, Ctrl+3 for copying first row columns
  if (e.ctrlKey && ['1', '2', '3'].includes(e.key) && !e.target.matches('input, textarea')) {
    e.preventDefault()
    const columnIndex = parseInt(e.key) - 1
    if (columnIndex < props.headers.length) {
      const firstRow = filteredAndSortedData.value[0]
      if (firstRow) {
        const header = props.headers[columnIndex]
        copyValue(firstRow[header], header)
      }
    }
  }
}

// Setup keyboard shortcuts
onMounted(() => {
  document.addEventListener('keydown', handleKeyboardShortcut)
})

onUnmounted(() => {
  document.removeEventListener('keydown', handleKeyboardShortcut)
  if (notificationTimeout) {
    clearTimeout(notificationTimeout)
  }
})

// Focus directive
const vFocus = {
  mounted: (el) => el.focus()
}

// Pagination
const pageSize = 10
const currentPage = ref(0)

const totalPages = computed(() => 
  Math.ceil(filteredAndSortedData.value.length / pageSize)
)

const paginatedData = computed(() => {
  const start = currentPage.value * pageSize
  return filteredAndSortedData.value.slice(start, start + pageSize)
})

const isEditing = (rowIndex, header) => {
  return editingCell.value && 
         editingCell.value.rowIndex === rowIndex && 
         editingCell.value.header === header
}

const startEditing = (rowIndex, header, currentValue) => {
  const globalRowIndex = rowIndex + currentPage.value * pageSize
  if (selectedRows.value.includes(globalRowIndex)) {
    editingCell.value = { rowIndex, header }
    editValue.value = currentValue
  }
}

const saveEdit = (currentRow) => {
  if (!editingCell.value) return

  const { rowIndex, header } = editingCell.value
  const globalRowIndex = rowIndex + currentPage.value * pageSize
  
  // Find the actual index in the original data array
  const originalIndex = props.data.findIndex(row => {
    return Object.entries(row).every(([key, value]) => 
      currentRow[key] === value
    )
  })

  if (originalIndex !== -1) {
    const newData = [...props.data]
    newData[originalIndex] = { ...newData[originalIndex], [header]: editValue.value }
    emit('update:data', newData)
  }

  editingCell.value = null
}

const cancelEdit = () => {
  editingCell.value = null
}

const sortColumn = ref('')
const sortDirection = ref('asc')
const searchQuery = ref('')
const selectedRows = ref([])

const sort = (header) => {
  if (sortColumn.value === header) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
  } else {
    sortColumn.value = header
    sortDirection.value = 'asc'
  }
}

const toggleRow = (index) => {
  const selectedIndex = selectedRows.value.indexOf(index)
  if (selectedIndex === -1) {
    selectedRows.value.push(index)
  } else {
    selectedRows.value.splice(selectedIndex, 1)
  }
  emit('selection-change', selectedRows.value)
}

const toggleAllRows = () => {
  const currentPageIndices = paginatedData.value.map((_, index) => 
    index + currentPage.value * pageSize
  )
  
  const allCurrentSelected = currentPageIndices.every(index => 
    selectedRows.value.includes(index)
  )
  
  if (allCurrentSelected) {
    selectedRows.value = selectedRows.value.filter(index => 
      !currentPageIndices.includes(index)
    )
  } else {
    const newSelected = new Set([...selectedRows.value, ...currentPageIndices])
    selectedRows.value = Array.from(newSelected)
  }
  
  emit('selection-change', selectedRows.value)
}

const allSelected = computed(() => {
  return paginatedData.value.length > 0 && 
         paginatedData.value.every((_, index) => 
           selectedRows.value.includes(index + currentPage.value * pageSize)
         )
})

const filteredData = computed(() => {
  if (!searchQuery.value || !features.value.search) return props.data

  const query = searchQuery.value.toLowerCase()
  const searchColumn = props.headers[1] // Search in second column by default
  
  return props.data.filter(row => {
    const value = String(row[searchColumn] || '').toLowerCase()
    return value.includes(query)
  })
})

const filteredAndSortedData = computed(() => {
  if (!sortColumn.value || !features.value.sort) return filteredData.value

  return [...filteredData.value].sort((a, b) => {
    const aVal = a[sortColumn.value]
    const bVal = b[sortColumn.value]

    if (typeof aVal === 'number' && typeof bVal === 'number') {
      return sortDirection.value === 'asc' ? aVal - bVal : bVal - aVal
    }

    if (typeof aVal === 'string' && aVal.startsWith('$') &&
        typeof bVal === 'string' && bVal.startsWith('$')) {
      const aNum = parseFloat(aVal.replace('$', ''))
      const bNum = parseFloat(bVal.replace('$', ''))
      return sortDirection.value === 'asc' ? aNum - bNum : bNum - aNum
    }

    return sortDirection.value === 'asc' 
      ? String(aVal).localeCompare(String(bVal))
      : String(bVal).localeCompare(String(aVal))
  })
})
</script>
