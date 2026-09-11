<script setup>
import { ref, onMounted, watch, computed } from 'vue'

const members = ref([])
const activities = ref([])
const newMemberName = ref('')

// Modalkontroller
const showMemberModal = ref(false)
const showActivityModal = ref(false)
const isEditing = ref(false)
const editingActivityId = ref(null)

// Standardvärden för formulär
const INITIAL_FORM = {
  memberName: '',
  day: 'Måndag',
  title: '',
  time: '',
  location: '',
  type: 'aktivitet',
  packingList: ''
}

const activityForm = ref({ ...INITIAL_FORM })

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

// STATISTIK
const totalActivities = computed(() => activities.value.length)
const completedActivities = computed(() => activities.value.filter(a => a.completed).length)

// Procentandel slutförda aktiviteter för visuell feedback
const completionPercentage = computed(() => {
  if (totalActivities.value === 0) return 0
  return Math.round((completedActivities.value / totalActivities.value) * 100)
})

// Sorterade aktiviteter per medlem
const sortedActivitiesByMember = computed(() => {
  const map = {}
  members.value.forEach(m => {
    map[m] = activities.value
      .filter(a => a.memberName === m)
      .sort((a, b) => {
        const dayDiff = (DAYS_ORDER[a.day] || 99) - (DAYS_ORDER[b.day] || 99)
        if (dayDiff !== 0) return dayDiff
        
        if (!a.time) return 1
        if (!b.time) return -1
        return a.time.localeCompare(b.time)
      })
  })
  return map
})

// LOCALSTORAGE
onMounted(() => {
  const savedMembers = localStorage.getItem('veckoplanerare_members')
  const savedActivities = localStorage.getItem('veckoplanerare_activities')

  members.value = savedMembers ? JSON.parse(savedMembers) : ['Harry', 'Rosa']
  activities.value = savedActivities ? JSON.parse(savedActivities) : [
    {
      id: 1,
      memberName: 'Harry',
      day: 'Torsdag',
      title: 'Trumpet',
      time: '17:00',
      location: 'Kulturhuset',
      type: 'idrott',
      packingList: 'Noter, instrument',
      completed: false
    },
    {
      id: 2,
      memberName: 'Rosa',
      day: 'Onsdag',
      title: 'Basketträning',
      time: '18:00',
      location: 'Rosendalsskolans gympasal',
      type: 'idrott',
      packingList: 'Träningskläder, skor, vattenflaska',
      completed: false
    }
  ]
})

watch(members, (newVal) => {
  localStorage.setItem('veckoplanerare_members', JSON.stringify(newVal))
}, { deep: true })

watch(activities, (newVal) => {
  localStorage.setItem('veckoplanerare_activities', JSON.stringify(newVal))
}, { deep: true })

watch([showMemberModal, showActivityModal], ([mOpen, aOpen]) => {
  document.body.classList.toggle('modal-open', mOpen || aOpen)
})

// MEDLEMSHANTERING
function addMember() {
  const name = newMemberName.value.trim()
  if (name && !members.value.includes(name)) {
    members.value.push(name)
    newMemberName.value = ''
  }
}

function removeMember(index) {
  const memberToRemove = members.value[index]
  members.value.splice(index, 1)
  activities.value = activities.value.filter(a => a.memberName !== memberToRemove)
}

function updateMemberName(oldName, newName, index) {
  const trimmed = newName.trim()
  if (!trimmed || members.value.includes(trimmed)) return
  
  members.value[index] = trimmed
  activities.value.forEach(act => {
    if (act.memberName === oldName) {
      act.memberName = trimmed
    }
  })
}

// AKTIVITETER
function openCreateModal(memberName = '') {
  isEditing.value = false
  editingActivityId.value = null
  activityForm.value = {
    ...INITIAL_FORM,
    memberName: memberName || members.value[0] || ''
  }
  showActivityModal.value = true
}

function openEditModal(act) {
  isEditing.value = true
  editingActivityId.value = act.id
  activityForm.value = { ...act }
  showActivityModal.value = true
}

function closeActivityModal() {
  showActivityModal.value = false
  isEditing.value = false
  editingActivityId.value = null
  activityForm.value = { ...INITIAL_FORM }
}

function saveActivity() {
  if (!activityForm.value.title || !activityForm.value.memberName) return

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

  closeActivityModal()
}

function removeActivity(id) {
  activities.value = activities.value.filter(a => a.id !== id)
}
</script>

<template>
  <div class="app-container">
    <header class="main-header">
      <div class="header-title">
        <h1>📅 weekPlanner</h1>
        <p>Aktivitetsöversikt & schema för familjen.</p>
      </div>

      <div class="header-actions">
        <button @click="showMemberModal = true" class="btn-secondary">
          Lägg till/ta bort familjemedlem
        </button>
        <button 
          @click="openCreateModal()" 
          class="btn-primary"
          :disabled="members.length === 0"
        >
          ➕ Ny aktivitet
        </button>
      </div>

      <!-- STATISTIK / KPI MED STYLE BINDING -->
      <div class="stats-container">
        <div class="stats-card">
          <span class="stats-label">Totalt</span>
          <span class="stats-value">{{ totalActivities }}</span>
        </div>
        <div 
          class="stats-card"
          :style="{ 
            backgroundColor: completionPercentage === 100 && totalActivities > 0 
              ? 'rgba(16, 185, 129, 0.35)' 
              : 'rgba(255, 255, 255, 0.15)' 
          }"
        >
          <span class="stats-label">Slutförda ({{ completionPercentage }}%)</span>
          <span class="stats-value">{{ completedActivities }} / {{ totalActivities }}</span>
        </div>
      </div>
    </header>

    <!-- SCHEMALAYOUT PER MEDLEM -->
    <main v-if="members.length > 0" class="list-section">
      <div class="columns-grid">
        <div v-for="member in members" :key="member" class="member-column card">
          <div class="column-header">
            <h3>👤 {{ member }}</h3>
            <button @click="openCreateModal(member)" class="btn-icon-add" title="Lägg till aktivitet">+</button>
          </div>

          <div class="column-content">
            <div 
              v-if="!sortedActivitiesByMember[member]?.length" 
              class="empty-column"
            >
              Inga aktiviteter planerade.
            </div>

            <div 
              v-for="act in sortedActivitiesByMember[member]" 
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

    <div v-else class="notice-box card">
      <p>Inga medlemmar/kolumner finns inlagda än.</p>
      <button @click="showMemberModal = true" class="btn-primary">Lägg till medlem/kolumn</button>
    </div>

    <!-- MODAL 1: HANTERA MEDLEMMAR -->
    <div v-if="showMemberModal" class="modal-overlay" @click.self="showMemberModal = false">
      <div class="modal-card">
        <div class="modal-header">
          <h2>Hantera Medlemmar & Kolumner</h2>
          <button @click="showMemberModal = false" class="btn-close">✕</button>
        </div>
        <div class="modal-body">
          <div class="add-member-form">
            <input 
              v-model="newMemberName" 
              type="text" 
              placeholder="Namn på medlem..."
              @keyup.enter="addMember"
            />
            <button @click="addMember" class="btn-primary">Lägg till</button>
          </div>

          <div v-if="members.length > 0" class="member-manage-list">
            <label class="form-label">Befintliga kolumner:</label>
            <div v-for="(member, index) in members" :key="index" class="rename-item">
              <input 
                :value="member" 
                @change="e => updateMemberName(member, e.target.value, index)"
                type="text"
              />
              <button @click="removeMember(index)" class="btn-delete" title="Ta bort">Ta bort</button>
            </div>
          </div>
        </div>
        <div class="modal-footer">
          <button @click="showMemberModal = false" class="btn-secondary">Stäng</button>
        </div>
      </div>
    </div>

    <!-- MODAL 2: SKAPA / REDIGERA AKTIVITET -->
    <div v-if="showActivityModal" class="modal-overlay" @click.self="closeActivityModal">
      <div class="modal-card">
        <div class="modal-header">
          <h2>{{ isEditing ? 'Redigera Aktivitet' : 'Skapa Ny Aktivitet' }}</h2>
          <button @click="closeActivityModal" class="btn-close">✕</button>
        </div>
        <form @submit.prevent="saveActivity">
          <div class="modal-body form-grid">
            <div class="form-control">
              <label>Medlem / Kolumn *</label>
              <select v-model="activityForm.memberName" required>
                <option v-for="member in members" :key="member" :value="member">
                  {{ member }}
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
            <button type="button" @click="closeActivityModal" class="btn-secondary">Avbryt</button>
            <button type="submit" class="btn-primary">
              {{ isEditing ? 'Spara ändringar' : 'Skapa aktivitet' }}
            </button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<style scoped>
@import './style.css';
</style>