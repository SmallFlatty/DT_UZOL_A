<script setup lang="ts">
import { computed, reactive, ref } from 'vue'
import { api } from '@/api'
import type { Parcel } from '@/types'
import { useParcels } from '@/composables/useParcels'
import { useToast } from '@/composables/useToast'
import { formatDateTime, formatWeight } from '@/format'
import StatusTrack from '@/components/StatusTrack.vue'

const { parcels, loaded, refresh } = useParcels()
const toast = useToast()

// ---------- prijatie zasielky ----------
const form = reactive({
  senderName: '',
  senderPhone: '',
  recipientName: '',
  recipientPhone: '',
  weight: null as number | null,
})
const saving = ref(false)
const lastCreated = ref<Parcel | null>(null)

const formValid = computed(
  () =>
    form.senderName.trim() &&
    form.senderPhone.trim() &&
    form.recipientName.trim() &&
    form.recipientPhone.trim() &&
    form.weight !== null &&
    form.weight > 0,
)

async function createParcel() {
  if (!formValid.value) return
  saving.value = true
  try {
    lastCreated.value = await api.createParcel({
      senderName: form.senderName.trim(),
      senderPhone: form.senderPhone.trim(),
      recipientName: form.recipientName.trim(),
      recipientPhone: form.recipientPhone.trim(),
      weight: form.weight!,
    })
    toast.ok(`Zásielka ${lastCreated.value.trackingNum} prijatá`)
    Object.assign(form, { senderName: '', senderPhone: '', recipientName: '', recipientPhone: '', weight: null })
    refresh()
  } catch (e) {
    toast.error(e)
  } finally {
    saving.value = false
  }
}

// ---------- zoznam ----------
const tab = ref<'ready' | 'all'>('ready')
const search = ref('')

const readyCount = computed(() => parcels.value.filter((p) => p.status === 'PRIPRAVENA').length)

const visible = computed(() => {
  const q = search.value.trim().toLowerCase()
  return parcels.value
    .filter((p) => tab.value === 'all' || p.status === 'PRIPRAVENA')
    .filter(
      (p) =>
        !q ||
        p.trackingNum.toLowerCase().includes(q) ||
        p.recipientName.toLowerCase().includes(q) ||
        p.recipientPhone.includes(q),
    )
})

const busyId = ref<string | null>(null)

async function handOver(p: Parcel) {
  busyId.value = p.id
  try {
    await api.handOver(p.id)
    toast.ok(`Zásielka ${p.trackingNum} vydaná`)
    if (lastCreated.value?.id === p.id) lastCreated.value = null
    refresh()
  } catch (e) {
    toast.error(e)
  } finally {
    busyId.value = null
  }
}

async function storno(p: Parcel) {
  if (!confirm(`Stornovať zásielku ${p.trackingNum}? Túto akciu nie je možné vrátiť.`)) return
  busyId.value = p.id
  try {
    await api.storno(p.id)
    toast.ok(`Zásielka ${p.trackingNum} stornovaná`)
    if (lastCreated.value?.id === p.id) lastCreated.value = null
    refresh()
  } catch (e) {
    toast.error(e)
  } finally {
    busyId.value = null
  }
}
</script>

<template>
  <main class="page layout">
    <aside class="side">
      <section class="panel">
        <h2>Prijať zásielku</h2>
        <form class="form" @submit.prevent="createParcel">
          <fieldset>
            <legend>Odosielateľ</legend>
            <div class="field">
              <label for="sn">Meno</label>
              <input id="sn" v-model="form.senderName" autocomplete="off" required />
            </div>
            <div class="field">
              <label for="sp">Telefón</label>
              <input id="sp" v-model="form.senderPhone" type="tel" required />
            </div>
          </fieldset>

          <fieldset>
            <legend>Príjemca</legend>
            <div class="field">
              <label for="rn">Meno</label>
              <input id="rn" v-model="form.recipientName" autocomplete="off" required />
            </div>
            <div class="field">
              <label for="rp">Telefón</label>
              <input id="rp" v-model="form.recipientPhone" type="tel" required />
            </div>
          </fieldset>

          <div class="field">
            <label for="w">Hmotnosť (kg)</label>
            <input id="w" v-model.number="form.weight" type="number" min="0.1" step="0.1" required />
          </div>

          <button class="btn btn--primary btn--full" :disabled="!formValid || saving">
            {{ saving ? 'Ukladám…' : 'Prijať zásielku' }}
          </button>
        </form>
      </section>

      <section v-if="lastCreated" class="label" aria-live="polite">
        <span class="label__caption">Napíšte na balík</span>
        <span class="label__num">{{ lastCreated.trackingNum }}</span>
        <span class="label__to">{{ lastCreated.recipientName }}, {{ lastCreated.recipientPhone }}</span>
      </section>
    </aside>

    <section class="panel list">
      <div class="list__head">
        <div class="tabs" role="tablist">
          <button role="tab" class="tab" :class="{ 'is-active': tab === 'ready' }" :aria-selected="tab === 'ready'" @click="tab = 'ready'">
            Na výdaj <span class="tab__count">{{ readyCount }}</span>
          </button>
          <button role="tab" class="tab" :class="{ 'is-active': tab === 'all' }" :aria-selected="tab === 'all'" @click="tab = 'all'">
            Všetky <span class="tab__count">{{ parcels.length }}</span>
          </button>
        </div>
        <input v-model="search" class="search" type="search" placeholder="Hľadať číslo, meno alebo telefón" aria-label="Hľadať" />
      </div>

      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>Zásielka</th>
              <th>Príjemca</th>
              <th>Hmotnosť</th>
              <th>Stav</th>
              <th><span class="sr-only">Akcie</span></th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="p in visible" :key="p.id">
              <td>
                <div class="tracking">{{ p.trackingNum }}</div>
                <div class="muted small">{{ formatDateTime(p.createdAt) }}</div>
              </td>
              <td>
                <div>{{ p.recipientName }}</div>
                <div class="muted small">{{ p.recipientPhone }}</div>
              </td>
              <td>{{ formatWeight(p.weight) }}</td>
              <td><StatusTrack :status="p.status" compact /></td>
              <td class="actions">
                <button
                  v-if="p.status === 'PRIPRAVENA'"
                  class="btn btn--dark"
                  :disabled="busyId === p.id"
                  @click="handOver(p)"
                >
                  Vydať
                </button>
                <span v-else class="muted small waiting">Ešte nedorazila</span>
                <button class="btn btn--danger" :disabled="busyId === p.id" @click="storno(p)">Stornovať</button>
              </td>
            </tr>
            <tr v-if="loaded && visible.length === 0">
              <td colspan="5" class="empty">
                <template v-if="search">Nič sa nenašlo. Skúste iné číslo alebo meno.</template>
                <template v-else-if="tab === 'ready'">Žiadna zásielka nečaká na výdaj. Nové sa objavia, keď ich depo doručí.</template>
                <template v-else>Zatiaľ žiadne zásielky. Prijmite prvú vo formulári vľavo.</template>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>
  </main>
</template>

<style scoped>
.layout {
  display: grid;
  grid-template-columns: 340px 1fr;
  gap: 24px;
  align-items: start;
}

.side {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.form {
  margin-top: 16px;
}

fieldset {
  border: none;
  padding: 0;
  margin: 0 0 6px;
}

legend {
  padding: 0;
  font-family: var(--font-display);
  font-size: 1.1rem;
  font-weight: 600;
  color: var(--ink-2);
  margin-bottom: 8px;
}

.btn--full {
  width: 100%;
  margin-top: 4px;
  padding-block: 12px;
}

/* stitok poslednej prijatej zasielky */
.label {
  background: var(--signal);
  border-radius: var(--radius);
  padding: 18px 20px;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.label__caption {
  font-size: 0.9rem;
  font-weight: 600;
}

.label__num {
  font-family: var(--font-display);
  font-weight: 700;
  font-size: 2.3rem;
  letter-spacing: 0.04em;
  line-height: 1.1;
}

.label__to {
  font-size: 0.95rem;
}

/* zoznam */
.list__head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
  flex-wrap: wrap;
  margin-bottom: 18px;
}

.tabs {
  display: flex;
  gap: 4px;
  background: var(--paper);
  padding: 4px;
  border-radius: 6px;
}

.tab {
  font: inherit;
  font-weight: 600;
  border: none;
  background: transparent;
  padding: 7px 14px;
  border-radius: 4px;
  cursor: pointer;
  color: var(--ink-2);
}

.tab.is-active {
  background: var(--surface);
  color: var(--ink);
}

.tab__count {
  margin-left: 6px;
  font-weight: 400;
  color: var(--muted);
}

.search {
  max-width: 300px;
}

.small {
  font-size: 0.85rem;
}

.actions {
  text-align: right;
  white-space: nowrap;
}

.waiting {
  display: inline-block;
  margin-right: 8px;
}

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
}

@media (max-width: 900px) {
  .layout {
    grid-template-columns: 1fr;
  }
  .search {
    max-width: none;
  }
}
</style>
