<script setup>
import { ref, onMounted, watch, computed } from 'vue'

// 1. REAKTIVA TILLSTÅND
const children = ref([])
const activities = ref([])
const newChildName = ref('')

// Modalkontroller
const showChildModal = ref(false)
const showActivityModal = ref(false)
const isEditing = ref(false)
const editingActivityId = ref(null)

// Aktivitet-formulär
const activityForm = ref({
  childName: '',
  day: 'Måndag',
  title: '',
  time: '',
  location: '',
  type: 'aktivitet',
  packingList: ''
})

// Veckodagarnas ordning för sortering
const DAYS_ORDER = {
  'Måndag': 1,
  'Tisdag': 2,
  'Onsdag': 3,
  'Torsdag': 4,
  'Fredag': 5,
  'Lördag': 6,
  'Söndag': 7
}

// 2. COMPUTED STATISTIK (Avanza KPI-stil)
const totalActivities = computed(() => activities.value.length)
const completedActivities = computed(() => activities.value.filter(a => a.completed).length)

// 3. LOCALSTORAGE
onMounted(() => {
  const savedChildren = localStorage.getItem('veckoplanerare_children')
  const savedActivities = localStorage.getItem('veckoplanerare_activities')

  if (savedChildren) {
    children.value = JSON.parse(savedChildren)
  } else {
    children.value = ['Saga', 'Albin']
  }

  if (savedActivities) {
    activities.value = JSON.parse(savedActivities)
  } else {
    activities.value = [
      {
        id: 1,
        childName: 'Saga',
        day: 'Tisdag',
        title: 'Fotbollsträning',
        time: '17:30',
        location: 'BIP IP',
        type: 'idrott',
        packingList: 'Gymnastikkläder, skor, vattenflaska',
        completed: false
      },
      {
        id: 2,
        childName: 'Albin',
        day: 'Torsdag',
        title: 'Skogsutflykt',
        time: '09:00',
        location: 'Naturreservatet',
        type: 'utflykt',
        packingList: 'Frukt, saft, varm tröja',
        completed: false
      }
    ]
  }
})

watch(children, (newVal) => {
  localStorage.setItem('veckoplanerare_children', JSON.stringify(newVal))
}, { deep: true })

watch(activities, (newVal) => {
  localStorage.setItem('veckoplanerare_activities', JSON.stringify(newVal))
}, { deep: true })

watch([showChildModal, showActivityModal], ([cOpen, aOpen]) => {
  if (cOpen || aOpen) {
    document.body.classList.add('modal-open')
  } else {
    document.body.classList.remove('modal-open')
  }
})

// 4. BARNHANTERING
function addChild() {
  const name = newChildName.value.trim()
  if (name && !children.value.includes(name)) {
    children.value.push(name)
    newChildName.value = ''
  }
}

function removeChild(index) {
  const childToRemove = children.value[index]
  children.value.splice(index, 1)
  activities.value = activities.value.filter(a => a.childName !== childToRemove)
}

function updateChildName(oldName, newName, index) {
  const trimmed = newName.trim()
  if (!trimmed) return
  
  children.value[index] = trimmed
  activities.value.forEach(act => {
    if (act.childName === oldName) {
      act.childName = trimmed
    }
  })
}

// 5. AKTIVITETER
function openCreateModal(childName = '') {
  isEditing.value = false
  editingActivityId.value = null
  activityForm.value = {
    childName: childName || (children.value[0] || ''),
    day: 'Måndag',
    title: '',
    time: '',
    location: '',
    type: 'aktivitet',
    packingList: ''
  }
  showActivityModal.value = true
}

function openEditModal(act) {
  isEditing.value = true
  editingActivityId.value = act.id
  activityForm.value = { ...act }
  showActivityModal.value = true
}

function saveActivity() {
  if (!activityForm.value.title || !activityForm.value.childName) return

  if (isEditing.value) {
    const idx = activities.value.findIndex(a => a.id === editingActivityId.value)
    if (idx !== -1) {
      activities.value[idx] = {
        ...activities.value[idx],
        ...activityForm.value
      }
    }
  } else {
    activities.value.push({
      id: Date.now(),
      ...activityForm.value,
      completed: false
    })
  }

  showActivityModal.value = false
}

function removeActivity(id) {
  activities.value = activities.value.filter(a => a.id !== id)
}

// 6. SORTERING (Dag -> Tid)
function getSortedActivitiesForChild(childName) {
  return activities.value
    .filter(a => a.childName === childName)
    .sort((a, b) => {
      const dayDiff = DAYS_ORDER[a.day] - DAYS_ORDER[b.day]
      if (dayDiff !== 0) return dayDiff
      
      if (!a.time) return 1
      if (!b.time) return -1
      return a.time.localeCompare(b.time)
    })
}
</script>

<template>
  <div class="app-container">
    <header class="main-header">
      <div>
        <h1>📅 weekPlanner</h1>
        <p>Aktivitetsöversikt & schema för familjen.</p>
      </div>

      <div class="header-actions">
        <button @click="showChildModal = true" class="btn-secondary">
          ⚙️ Hantera barn
        </button>
        <button 
          @click="openCreateModal()" 
          class="btn-primary"
          :disabled="children.length === 0"
        >
          ➕ Ny aktivitet
        
        </button>

        <!-- STATISTIK -->
        <p>Totalt antal aktiviteter:</p>
        <p>{{ totalActivities }}</p>
        <p>Antal slutförda aktiviteter:</p>
        <p>{{ completedActivities }}</p>

      </div>
    </header>

    <!-- SCHEMALAYOUT PER BARN -->
    <main v-if="children.length > 0" class="card list-section">
      <div 
        class="columns-grid" 
        :style="{ gridTemplateColumns: `repeat(${children.length}, minmax(280px, 1fr))` }"
      >
        <div v-for="child in children" :key="child" class="child-column">
          <div class="column-header">
            <h3>👤 {{ child }}</h3>
            <button @click="openCreateModal(child)" class="btn-icon-add" title="Lägg till aktivitet">+</button>
          </div>

          <div class="column-content">
            <div 
              v-if="getSortedActivitiesForChild(child).length === 0" 
              class="empty-column"
            >
              Inga aktiviteter planerade.
            </div>

            <div 
              v-for="act in getSortedActivitiesForChild(child)" 
              :key="act.id"
              class="activity-card"
              :class="[act.type, { 'is-completed': act.completed }]"
            >
              <div class="card-header">
                <span class="badge day-badge">{{ act.day }}</span>
                <button @click="openEditModal(act)" class="btn-edit-small">Redigera</button>
              </div>

              <h4>{{ act.title }}</h4>

              <p v-if="act.time || act.location" class="meta-info">
                <span v-if="act.time">Tid: {{ act.time }}</span>
                <span v-if="act.location">Plats: {{ act.location }}</span>
              </p>

              <div v-if="act.packingList" class="packing-box">
                <strong>Packa:</strong> {{ act.packingList }}
              </div>

              <div class="card-footer">
                <label class="checkbox-label">
                  <input type="checkbox" v-model="act.completed" />
                  <span>Klar</span>
                </label>
                <button @click="removeActivity(act.id)" class="btn-delete">Ta bort</button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </main>

    <div v-else class="notice-box">
      <p>Inga barn/kolumner finns inlagda än.</p>
      <button @click="showChildModal = true" class="btn-primary">Lägg till barn/kolumn</button>
    </div>

    <!-- MODAL 1: HANTERA BARN -->
    <div v-if="showChildModal" class="modal-overlay" @click.self="showChildModal = false">
      <div class="modal-card">
        <div class="modal-header">
          <h2>Hantera Barn & Kolumner</h2>
          <button @click="showChildModal = false" class="btn-close">✕</button>
        </div>
        <div class="modal-body">
          <div class="add-child-form">
            <input 
              v-model="newChildName" 
              type="text" 
              placeholder="Namn på barn..."
              @keyup.enter="addChild"
            />
            <button @click="addChild" class="btn-primary">Lägg till</button>
          </div>

          <div v-if="children.length > 0" class="child-manage-list">
            <label class="form-label">Befintliga kolumner:</label>
            <div v-for="(child, index) in children" :key="index" class="rename-item">
              <input 
                :value="child" 
                @change="e => updateChildName(child, e.target.value, index)"
                type="text"
              />
              <button @click="removeChild(index)" class="btn-delete" title="Ta bort">Ta bort</button>
            </div>
          </div>
        </div>
        <div class="modal-footer">
          <button @click="showChildModal = false" class="btn-secondary">Stäng</button>
        </div>
      </div>
    </div>

    <!-- MODAL 2: SKAPA / REDIGERA AKTIVITET -->
    <div v-if="showActivityModal" class="modal-overlay" @click.self="showActivityModal = false">
      <div class="modal-card">
        <div class="modal-header">
          <h2>{{ isEditing ? 'Redigera Aktivitet' : 'Skapa Ny Aktivitet' }}</h2>
          <button @click="showActivityModal = false" class="btn-close">✕</button>
        </div>
        <form @submit.prevent="saveActivity">
          <div class="modal-body form-grid">
            <div class="form-control">
              <label>Barn / Kolumn *</label>
              <select v-model="activityForm.childName" required>
                <option v-for="child in children" :key="child" :value="child">
                  {{ child }}
                </option>
              </select>
            </div>

            <div class="form-control">
              <label>Dag *</label>
              <select v-model="activityForm.day">
                <option>Måndag</option>
                <option>Tisdag</option>
                <option>Onsdag</option>
                <option>Torsdag</option>
                <option>Fredag</option>
                <option>Lördag</option>
                <option>Söndag</option>
              </select>
            </div>

            <div class="form-control full-width">
              <label>Aktivitet / Rubrik *</label>
              <input v-model="activityForm.title" placeholder="t.ex. Fotbollsträning" required />
            </div>

            <div class="form-control">
              <label>Tid</label>
              <input v-model="activityForm.time" type="time" />
            </div>

            <div class="form-control">
              <label>Plats</label>
              <input v-model="activityForm.location" placeholder="t.ex. Sporthallen" />
            </div>

            <div class="form-control full-width">
              <label>Kategori</label>
              <select v-model="activityForm.type">
                <option value="idrott">Idrott / Träning</option>
                <option value="utflykt">Utflykt / Fika</option>
                <option value="läxa">Läxa / Skola</option>
                <option value="aktivitet">Övrigt</option>
              </select>
            </div>

            <div class="form-control full-width">
              <label>Packlista / Kom ihåg</label>
              <input v-model="activityForm.packingList" placeholder="t.ex. Skor, vattenflaska" />
            </div>
          </div>
          <div class="modal-footer">
            <button type="button" @click="showActivityModal = false" class="btn-secondary">Avbryt</button>
            <button type="submit" class="btn-primary">
              {{ isEditing ? 'Spara ändringar' : 'Skapa aktivitet' }}
            </button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<style>
@import './style.css';
</style>