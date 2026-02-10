<script setup>
import { ref, onMounted, watch, computed } from 'vue'
import axios from 'axios'

/* =========================
   CONFIG
========================= */
const API_BASE = import.meta.env.VITE_API_BASE_URL

/* =========================
   STATE
========================= */
const items = ref([])
const totalItems = ref(0)
const loading = ref(false)

const page = ref(1)
const itemsPerPage = ref(100)

/* FILTERS */
const filters = ref({})

/* =========================
   HEADERS
========================= */
const headers = [
  { title: 'Date & Time', key: 'createdDateTime' },
  { title: 'ID', key: 'id' },
  { title: 'User Principal Name', key: 'userPrincipalName' },
  { title: 'Application', key: 'appDisplayName' },
  { title: 'IP Address', key: 'ipaddress' },
  { title: 'Client', key: 'clientAppUsed' },
  { title: 'Resource', key: 'resourceDisplayName' },
  { title: 'Resource ID', key: 'resourceId' },
  { title: 'User', key: 'userDisplayName' }
]

/* =========================
   HELPERS
========================= */
const formatDateTime = (value) => {
  if (!value) return '—'
  const d = new Date(value)
  return d.toLocaleString()
}

const hasFilters = computed(() =>
  Object.values(filters.value).some(v => v && v.trim() !== '')
)

/* =========================
   FETCH DATA
========================= */
const fetchLogs = async () => {
  loading.value = true

  try {
    let response

    if (hasFilters.value) {
      // POST FILTER API
      const payload = Object.entries(filters.value)
        .filter(([_, v]) => v && v.trim() !== '')
        .map(([key, value]) => ({
          columnName: key,
          operator: 'contains',
          value
        }))

      response = await axios.post(
        `${API_BASE}/api/signin-logs/filter`,
        payload,
        {
          params: {
            page: page.value,
            pageSize: itemsPerPage.value
          }
        }
      )
    } else {
      // GET PAGED API
      response = await axios.get(
        `${API_BASE}/api/signin-logs`,
        {
          params: {
            page: page.value,
            pageSize: itemsPerPage.value
          }
        }
      )
    }

    // 🔒 SAFETY CHECK
    if (!Array.isArray(response.data?.data)) {
      console.warn('INVALID API RESPONSE', response.data)
      items.value = []
      totalItems.value = 0
      return
    }

    items.value = response.data.data
    totalItems.value = response.data.totalCount ?? response.data.data.length

    console.log('Rows loaded:', items.value.length)
    console.log('Total rows:', totalItems.value)

  } catch (err) {
    console.error('API ERROR:', err)
    items.value = []
    totalItems.value = 0
  } finally {
    loading.value = false
  }
}

/* =========================
   WATCHERS
========================= */
watch([page, itemsPerPage], fetchLogs)

onMounted(fetchLogs)
</script>

<template>
  <v-container fluid class="pa-6">
    <v-card rounded="lg" elevation="2">
      <!-- HEADER -->
      <v-card-title class="d-flex justify-space-between">
        <div>
          <h3>Audit Sign-In Logs</h3>
          <div class="text-caption text-grey">
            Server-side pagination · Filters · Azure API
          </div>
        </div>
        <v-chip color="primary" variant="tonal">
          {{ totalItems.toLocaleString() }} records
        </v-chip>
      </v-card-title>

      <v-divider />

      <!-- TABLE -->
      <v-data-table-server
        :headers="headers"
        :items="items"
        :items-length="totalItems"
        :loading="loading"
        :page="page"
        :items-per-page="itemsPerPage"
        @update:page="page = $event"
        @update:items-per-page="itemsPerPage = $event"
      >
        <!-- FILTER HEADER -->
        <template #headers="{ columns }">
          <tr>
            <th v-for="col in columns" :key="col.key">
              <div class="d-flex align-center justify-space-between">
                <span>{{ col.title }}</span>
                <v-menu :close-on-content-click="false">
                  <template #activator="{ props }">
                    <v-btn icon size="x-small" v-bind="props">
                      <v-icon>mdi-filter-outline</v-icon>
                    </v-btn>
                  </template>

                  <v-card class="pa-3" width="240">
                    <v-text-field
                      v-model="filters[col.key]"
                      density="compact"
                      label="Filter"
                      clearable
                      @keyup.enter="fetchLogs"
                    />
                    <v-btn block color="primary" @click="fetchLogs">
                      Apply
                    </v-btn>
                  </v-card>
                </v-menu>
              </div>
            </th>
          </tr>
        </template>

        <!-- CELLS -->
        <template #item="{ item }">
  {{ formatDateTime(item.createdDateTime) }}
</template>

        <!-- NO DATA -->
        <template #no-data>
          <div class="pa-8 text-center text-grey">
            No audit logs found
          </div>
        </template>
      </v-data-table-server>
    </v-card>
  </v-container>
</template>
