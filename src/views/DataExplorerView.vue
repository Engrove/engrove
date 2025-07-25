<!-- src/views/DataExplorerView.vue -->
<script setup>
import { computed, ref, watch, onMounted } from 'vue'; // Importera onMounted
import { useHead } from '@unhead/vue';
import { useExplorerStore } from '@/store/explorerStore.js';
import { useTonearmStore } from '@/store/tonearmStore.js';   // Importera
import { useEstimatorStore } from '@/store/estimatorStore.js'; // Importera
import ResultsTable from '@/components/ResultsTable.vue';
import RangeFilter from '@/components/RangeFilter.vue';
import ItemDetailModal from '@/components/ItemDetailModal.vue';

const store = useExplorerStore();
const tonearmStore = useTonearmStore();     // Skapa instans
const estimatorStore = useEstimatorStore(); // Skapa instans

useHead({
  title: 'Component Database Explorer | Engrove Audio Toolkit',
  meta: [
    { 
      name: 'description', 
      content: 'Search and filter a comprehensive database of tonearms and phono cartridges. Find specifications, compliance data, effective mass, and more.' 
    },
    { property: 'og:title', content: 'Component Database Explorer | Engrove Audio Toolkit' },
    { property: 'og:description', content: 'Search and filter a comprehensive database of tonearms and phono cartridges.' },
  ],
});

const isModalOpen = ref(false);
const selectedItem = ref(null);

// KORRIGERING: Använd onMounted för att hantera initieringen.
onMounted(async () => {
  store.isLoading = true; // Sätt laddningsstatus i explorerStore
  try {
    // Vänta på att båda datakällorna är klara
    await Promise.all([
      tonearmStore.initialize(),
      estimatorStore.initialize()
    ]);
    // Initiera sedan explorerStore, som nu vet att datan finns
    await store.initialize();
  } catch (error) {
    store.error = error.message;
    console.error("Failed to initialize Data Explorer:", error);
  } finally {
    store.isLoading = false;
  }
});

function showItemDetails(item) {
  selectedItem.value = item;
  isModalOpen.value = true;
}

watch(() => store.availableNumericFilters, (newFilters) => {
  newFilters.forEach(filter => {
    if (!store.numericFilters[filter.key]) {
      store.numericFilters[filter.key] = { min: null, max: null };
    }
  });
}, { immediate: true, deep: true });

const cartridgeHeaders = [
  { key: 'manufacturer', label: 'Manufacturer', sortable: true }, { key: 'model', label: 'Model', sortable: true }, { key: 'type', label: 'Type', sortable: true },
  { key: 'cu_dynamic_10hz', label: 'Compliance @ 10Hz', sortable: true }, { key: 'weight_g', label: 'Weight (g)', sortable: true }, { key: 'stylus_family', label: 'Stylus', sortable: true }
];
const tonearmHeaders = [
  { key: 'manufacturer', label: 'Manufacturer', sortable: true }, { key: 'model', label: 'Model', sortable: true }, { key: 'effective_mass_g', label: 'Effective Mass (g)', sortable: true },
  { key: 'effective_length_mm', label: 'Length (mm)', sortable: true }, { key: 'bearing_type', label: 'Bearing', sortable: true }, { key: 'arm_shape', label: 'Shape', sortable: true }
];
const currentHeaders = computed(() => store.dataType === 'cartridges' ? cartridgeHeaders : tonearmHeaders);
</script>

<template>
  <div class="tool-view">
    <div class="tool-header">
      <h1>Data Explorer</h1>
    </div>
    <p class="tool-description">
      Search, filter, and explore the complete database of tonearms and cartridges. Use the controls to start a search.
    </p>

    <div v-if="store.isLoading" class="status-container">Loading databases...</div>
    <div v-else-if="store.error" class="status-container error">{{ store.error }}</div>
    
    <div v-else class="explorer-layout">
      <aside class="filter-panel">
        <h3>Controls</h3>
        <div class="control-group">
          <label>1. Select Data Type</label>
          <div class="button-group">
            <button @click="store.setDataType('tonearms')" :class="{ active: store.dataType === 'tonearms' }">Tonearms</button>
            <button @click="store.setDataType('cartridges')" :class="{ active: store.dataType === 'cartridges' }">Cartridges</button>
          </div>
        </div>

        <div v-if="store.dataType" class="filter-controls">
          <label>2. Filter Results</label>

          <div class="control-group">
            <input type="text" placeholder="Search by name..." v-model="store.searchTerm" @input="store.currentPage = 1" class="search-input">
          </div>
          
          <div v-for="filter in store.availableFilters" :key="filter.key" class="control-group">
            <label :for="filter.key">{{ filter.name }}</label>
            <select :id="filter.key" v-model="store.filters[filter.key]" @change="store.currentPage = 1" class="filter-select">
              <option :value="undefined">All</option>
              <option v-for="option in filter.options" :key="option.id" :value="option.id">{{ option.name }}</option>
            </select>
          </div>

          <div v-for="filter in store.availableNumericFilters" :key="filter.key" class="control-group">
            <RangeFilter
              :label="filter.label"
              :unit="filter.unit"
              :modelValue="store.numericFilters[filter.key] || { min: null, max: null }"
              @update:modelValue="newValue => store.updateNumericFilter(filter.key, newValue)"
            />
          </div>

          <button @click="store.resetFilters" class="reset-filters-btn">Reset All Filters</button>
        </div>
      </aside>

      <main class="results-area">
        <div v-if="store.totalResultsCount === 0 && store.searchTerm === '' && Object.values(store.filters).every(v => !v) && Object.values(store.numericFilters).every(v => v.min === null && v.max === null)" class="results-placeholder">
          <svg xmlns="http://www.w3.org/2000/svg" width="64" height="64" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1" stroke-linecap="round" stroke-linejoin="round"><path d="M20 17.58A5 5 0 0 0 15 8l-6.4 6.4a5 5 0 1 0 7.8 7.8L20 17.58z"></path></svg>
          <p>Use the filters to begin your search.</p>
        </div>

        <div v-else>
          <div class="results-header">
            <h3>Found {{ store.totalResultsCount }} {{ store.dataType }}</h3>
            <button @click="store.exportToCSV" class="download-csv-btn" :disabled="store.totalResultsCount === 0">
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path><polyline points="7 10 12 15 17 10"></polyline><line x1="12" y1="15" x2="12" y2="3"></line></svg>
              Download CSV
            </button>
          </div>
          
          <div v-if="store.totalResultsCount > store.itemsPerPage" class="pagination-controls pagination-top">
            <button @click="store.prevPage()" :disabled="!store.canGoPrev">‹ Previous</button>
            <span>Page {{ store.currentPage }} of {{ Math.ceil(store.totalResultsCount / store.itemsPerPage) }}</span>
            <button @click="store.nextPage()" :disabled="!store.canGoNext">Next ›</button>
          </div>
          
          <ResultsTable 
            :items="store.paginatedResults" 
            :headers="currentHeaders"
            :sort-key="store.sortKey"
            :sort-order="store.sortOrder"
            @sort="store.setSortKey"
            @row-click="showItemDetails"
          />

          <div v-if="store.totalResultsCount > store.itemsPerPage" class="pagination-controls pagination-bottom">
            <button @click="store.prevPage()" :disabled="!store.canGoPrev">‹ Previous</button>
            <span>Page {{ store.currentPage }} of {{ Math.ceil(store.totalResultsCount / store.itemsPerPage) }}</span>
            <button @click="store.nextPage()" :disabled="!store.canGoNext">Next ›</button>
          </div>
        </div>
      </main>
    </div>
    
    <ItemDetailModal 
      :isOpen="isModalOpen" 
      :item="selectedItem" 
      :dataType="store.dataType"
      @close="isModalOpen = false" 
    />
  </div>
</template>

<style scoped>
.tool-view { display: flex; flex-direction: column; }
.tool-header { padding-bottom: 1rem; margin-bottom: 0.5rem; border-bottom: 1px solid var(--border-color); }
.tool-header h1 { margin: 0; font-size: 1.75rem; color: var(--header-color); }
.tool-description { margin-top: 0; margin-bottom: 2rem; color: var(--label-color); max-width: 80ch; }
.explorer-layout { display: grid; grid-template-columns: 280px minmax(0, 1fr); gap: 2rem; align-items: flex-start; }
.filter-panel { background: var(--panel-bg); padding: 1.5rem; border-radius: 8px; border: 1px solid var(--border-color); position: sticky; top: 2rem; }
.filter-panel h3 { margin-top: 0; color: var(--header-color); border-bottom: 1px solid var(--border-color); padding-bottom: 0.75rem; margin-bottom: 1.5rem; }
.control-group { margin-bottom: 1.5rem; }
.control-group label, .filter-controls > label { display: block; font-weight: 600; color: var(--label-color); margin-bottom: 0.75rem; font-size: 0.9rem; text-transform: uppercase; letter-spacing: 0.5px; }
.button-group { display: flex; width: 100%; }
.button-group button { flex-grow: 1; padding: 0.75rem 0.5rem; font-size: 1rem; font-weight: 600; background-color: #fff; border: 1px solid var(--border-color); cursor: pointer; transition: all 0.2s ease; color: var(--accent-color); }
.button-group button:first-child { border-top-left-radius: 6px; border-bottom-left-radius: 6px; }
.button-group button:last-child { border-top-right-radius: 6px; border-bottom-right-radius: 6px; border-left-width: 0; }
.button-group button.active { background-color: var(--accent-color); color: white; border-color: var(--accent-color); z-index: 2; }
.search-input, .filter-select { width: 100%; padding: 0.5rem 0.75rem; font-size: 1rem; background-color: #fff; border: 1px solid #ced4da; border-radius: 4px; box-sizing: border-box; }
.reset-filters-btn { width: 100%; padding: 0.6rem; font-size: 0.9rem; font-weight: 600; color: var(--danger-text); background-color: transparent; border: 1px solid #f5c6cb; border-radius: 6px; cursor: pointer; transition: all 0.2s ease; margin-top: 1rem; }
.reset-filters-btn:hover { background-color: var(--danger-color); color: white; border-color: var(--danger-text); }
.results-area { min-height: 500px; }
.results-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1rem; }
.results-header h3 { margin: 0; color: var(--header-color); }
.pagination-controls { display: flex; justify-content: center; align-items: center; gap: 1rem; }
.pagination-controls span { font-size: 0.9rem; color: var(--label-color); font-weight: 500;}
.pagination-top { margin-bottom: 1rem; }
.pagination-bottom { margin-top: 1.5rem; }
.pagination-controls button { padding: 0.5rem 1rem; font-weight: 600; background-color: #fff; border: 1px solid var(--border-color); border-radius: 4px; cursor: pointer; transition: all 0.2s ease; }
.pagination-controls button:hover:not(:disabled) { background-color: var(--panel-bg); color: var(--accent-color); }
.pagination-controls button:disabled { opacity: 0.5; cursor: not-allowed; }
.results-placeholder { display: flex; flex-direction: column; justify-content: center; align-items: center; height: 400px; border: 2px dashed var(--border-color); border-radius: 8px; color: var(--label-color); }
.results-placeholder svg { color: #bdc3c7; margin-bottom: 1rem; }
.results-placeholder p { font-size: 1.25rem; font-weight: 500; }
.status-container { padding: 2rem; text-align: center; background-color: var(--panel-bg); border-radius: 6px; }
.status-container.error { background-color: var(--danger-color); color: var(--danger-text); }
.download-csv-btn { display: inline-flex; align-items: center; gap: 0.5rem; padding: 0.5rem 1rem; font-size: 0.9rem; font-weight: 600; color: #fff; background-color: #28a745; border: 1px solid #28a745; border-radius: 6px; cursor: pointer; transition: all 0.2s ease; }
.download-csv-btn:hover:not(:disabled) { background-color: #218838; border-color: #1e7e34; }
.download-csv-btn:disabled { background-color: #6c757d; border-color: #6c757d; opacity: 0.65; cursor: not-allowed; }
@media (max-width: 900px) { .explorer-layout { grid-template-columns: 1fr; } .filter-panel { position: static; } }
</style>
