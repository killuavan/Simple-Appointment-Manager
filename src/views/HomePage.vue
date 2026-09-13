<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Simple Appointment Manager</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
        <main class="page-shell">
          <section class="intro">
            <div>
              <p class="eyebrow">Appointment desk</p>
              <h1>Keep every visit on time.</h1>
              <p class="intro-copy">Add, review, and update your client appointments in one calm, focused place.</p>
            </div>
            <div class="summary" aria-label="Appointment summary">
              <span>{{ appointments.length }}</span>
              <small>Total appointments</small>
            </div>
          </section>

          <section class="workspace">
            <form class="appointment-form" @submit.prevent="addAppointment">
              <div class="section-heading">
                <div>
                  <p class="eyebrow">New entry</p>
                  <h2>Add appointment</h2>
                </div>
                <ion-icon :icon="calendarOutline" aria-hidden="true" />
              </div>

              <ion-item class="field" lines="none">
                <ion-input v-model="form.client" label="Client name" label-placement="stacked" placeholder="e.g. Maria Santos" required />
              </ion-item>
              <div class="field-row">
                <ion-item class="field" lines="none">
                  <ion-input v-model="form.date" label="Date" label-placement="stacked" type="date" required />
                </ion-item>
                <ion-item class="field" lines="none">
                  <ion-input v-model="form.time" label="Time" label-placement="stacked" type="time" required />
                </ion-item>
              </div>
              <ion-item class="field" lines="none">
                <ion-textarea v-model="form.purpose" label="Purpose" label-placement="stacked" placeholder="What is this appointment for?" :auto-grow="true" required />
              </ion-item>
              <ion-item class="field" lines="none">
                <ion-select v-model="form.status" label="Status" label-placement="stacked" interface="popover">
                  <ion-select-option v-for="status in statuses" :key="status" :value="status">{{ status }}</ion-select-option>
                </ion-select>
              </ion-item>
              <ion-button class="add-button" type="submit" expand="block">
                <ion-icon slot="start" :icon="addOutline" aria-hidden="true" />
                Add appointment
              </ion-button>
            </form>

            <section class="appointment-list" aria-labelledby="upcoming-heading">
              <div class="section-heading list-heading">
                <div>
                  <p class="eyebrow">Your schedule</p>
                  <h2 id="upcoming-heading">Appointments</h2>
                </div>
                <ion-select v-model="filter" class="filter-select" interface="popover" aria-label="Filter appointments">
                  <ion-select-option value="All">All</ion-select-option>
                  <ion-select-option v-for="status in statuses" :key="status" :value="status">{{ status }}</ion-select-option>
                </ion-select>
              </div>

              <div v-if="filteredAppointments.length" class="cards">
                <article v-for="appointment in filteredAppointments" :key="appointment.id" class="appointment-card">
                  <div class="card-topline">
                    <span class="date-label">{{ formatDate(appointment.date) }} · {{ formatTime(appointment.time) }}</span>
                    <ion-button fill="clear" class="delete-button" aria-label="Delete appointment" @click="removeAppointment(appointment.id)">
                      <ion-icon slot="icon-only" :icon="trashOutline" aria-hidden="true" />
                    </ion-button>
                  </div>
                  <h3>{{ appointment.client }}</h3>
                  <p>{{ appointment.purpose }}</p>
                  <ion-select v-model="appointment.status" class="status-select" interface="popover" aria-label="Appointment status" @ion-change="saveAppointments">
                    <ion-select-option v-for="status in statuses" :key="status" :value="status">{{ status }}</ion-select-option>
                  </ion-select>
                </article>
              </div>
              <div v-else class="empty-state">
                <ion-icon :icon="calendarClearOutline" aria-hidden="true" />
                <h3>No appointments yet</h3>
                <p>Your added appointments will appear here.</p>
              </div>
            </section>
          </section>
        </main>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { computed, onMounted, reactive, ref } from 'vue';
import { IonButton, IonContent, IonHeader, IonIcon, IonInput, IonItem, IonPage, IonSelect, IonSelectOption, IonTextarea, IonTitle, IonToolbar } from '@ionic/vue';
import { addOutline, calendarClearOutline, calendarOutline, trashOutline } from 'ionicons/icons';
import { database, isFirebaseConfigured } from '@/firebase';
import { auth } from '@/firebase';
import { signInAnonymously } from 'firebase/auth';
import { onValue, ref as databaseRef, remove, set } from 'firebase/database';

type AppointmentStatus = 'Scheduled' | 'Confirmed' | 'Completed' | 'Cancelled';
type Appointment = { id: number; client: string; date: string; time: string; purpose: string; status: AppointmentStatus };

const statuses: AppointmentStatus[] = ['Scheduled', 'Confirmed', 'Completed', 'Cancelled'];
const appointments = ref<Appointment[]>([]);
const filter = ref('All');
const form = reactive({ client: '', date: '', time: '', purpose: '', status: 'Scheduled' as AppointmentStatus });

const filteredAppointments = computed(() => {
  return appointments.value
    .filter((appointment) => filter.value === 'All' || appointment.status === filter.value)
    .sort((first, second) => `${first.date}${first.time}`.localeCompare(`${second.date}${second.time}`));
});

onMounted(async () => {
  if (isFirebaseConfigured && database && auth) {
    await signInAnonymously(auth);
    onValue(databaseRef(database, 'appointments'), (snapshot) => {
      const data = snapshot.val() as Record<string, Appointment> | null;
      appointments.value = data ? Object.values(data) : [];
    });
  } else {
    appointments.value = loadAppointments();
  }
});

function loadAppointments(): Appointment[] {
  try {
    return JSON.parse(localStorage.getItem('appointments') || '[]') as Appointment[];
  } catch {
    return [];
  }
}

async function saveAppointments() {
  if (isFirebaseConfigured) {
    if (!database) return;
    const firebaseDatabase = database;
    await Promise.all(appointments.value.map((appointment) => set(databaseRef(firebaseDatabase, `appointments/${appointment.id}`), appointment)));
  } else {
  localStorage.setItem('appointments', JSON.stringify(appointments.value));
  }
}

async function addAppointment() {
  const appointment = { id: Date.now(), ...form };
  appointments.value.push(appointment);
  if (isFirebaseConfigured && database) {
    await set(databaseRef(database, `appointments/${appointment.id}`), appointment);
  }
  saveAppointments();
  form.client = '';
  form.date = '';
  form.time = '';
  form.purpose = '';
  form.status = 'Scheduled';
}

async function removeAppointment(id: number) {
  appointments.value = appointments.value.filter((appointment) => appointment.id !== id);
  if (isFirebaseConfigured && database) {
    await remove(databaseRef(database, `appointments/${id}`));
  } else {
    saveAppointments();
  }
}

function formatDate(date: string) {
  return new Intl.DateTimeFormat('en-US', { month: 'short', day: 'numeric', year: 'numeric' }).format(new Date(`${date}T00:00:00`));
}

function formatTime(time: string) {
  return new Intl.DateTimeFormat('en-US', { hour: 'numeric', minute: '2-digit' }).format(new Date(`1970-01-01T${time}`));
}
</script>

<style scoped>
:global(body) { background: #f4f1eb; }
.app-header ion-toolbar { --background: #173c3a; --color: #f8f4ed; --border-width: 0; }
.page-shell { max-width: 1180px; margin: 0 auto; padding: 40px 22px 64px; }
.intro { display: flex; align-items: end; justify-content: space-between; gap: 24px; margin-bottom: 34px; }
.eyebrow { color: #c66d4e; font-size: 11px; font-weight: 800; letter-spacing: 1.4px; margin: 0 0 9px; text-transform: uppercase; }
h1, h2, h3, p { margin-top: 0; }
h1 { color: #173c3a; font-family: Georgia, serif; font-size: clamp(34px, 6vw, 62px); font-weight: 500; letter-spacing: 0; line-height: .98; margin-bottom: 14px; max-width: 590px; }
.intro-copy { color: #68716e; font-size: 16px; line-height: 1.55; margin-bottom: 0; max-width: 510px; }
.summary { background: #dce6dc; border-radius: 4px; color: #173c3a; min-width: 140px; padding: 19px 21px; }
.summary span { display: block; font-family: Georgia, serif; font-size: 36px; line-height: 1; }
.summary small { color: #65756b; display: block; font-size: 12px; margin-top: 7px; }
.workspace { align-items: start; display: grid; gap: 26px; grid-template-columns: minmax(280px, 380px) 1fr; }
.appointment-form, .appointment-list { background: #fffdfa; border: 1px solid #e3ded4; border-radius: 6px; padding: 25px; }
.section-heading { align-items: center; display: flex; justify-content: space-between; margin-bottom: 18px; }
.section-heading h2 { color: #173c3a; font-family: Georgia, serif; font-size: 27px; font-weight: 500; margin-bottom: 0; }
.section-heading > ion-icon { color: #c66d4e; font-size: 28px; }
.field { --background: #f4f1eb; --border-radius: 3px; --padding-start: 13px; --inner-padding-end: 13px; margin-bottom: 10px; }
.field ion-input, .field ion-textarea, .field ion-select { --highlight-color: #c66d4e; font-size: 14px; }
.field-row { display: grid; gap: 10px; grid-template-columns: 1fr 1fr; }
.add-button { --background: #c66d4e; --background-activated: #a9533b; --border-radius: 3px; --box-shadow: none; height: 48px; margin: 17px 0 0; text-transform: none; }
.list-heading { margin-bottom: 16px; }
.filter-select { --background: #eef1eb; --border-radius: 3px; --padding-end: 11px; --padding-start: 11px; color: #31514b; font-size: 13px; min-width: 88px; }
.cards { display: grid; gap: 12px; }
.appointment-card { background: #f7f5ef; border-left: 4px solid #c66d4e; padding: 17px 18px 15px; }
.card-topline { align-items: center; display: flex; justify-content: space-between; }
.date-label { color: #c66d4e; font-size: 12px; font-weight: 700; }
.delete-button { --color: #87908a; height: 28px; margin: -5px -10px 0 0; width: 28px; }
.appointment-card h3 { color: #173c3a; font-family: Georgia, serif; font-size: 22px; font-weight: 500; margin-bottom: 6px; }
.appointment-card p { color: #68716e; font-size: 14px; line-height: 1.45; margin-bottom: 10px; }
.status-select { --background: #dce6dc; --border-radius: 3px; --padding-end: 9px; --padding-start: 9px; color: #31514b; font-size: 12px; font-weight: 700; max-width: 135px; }
.empty-state { align-items: center; border: 1px dashed #c9c7bf; color: #7c827d; display: flex; flex-direction: column; justify-content: center; min-height: 245px; padding: 25px; text-align: center; }
.empty-state ion-icon { color: #c66d4e; font-size: 34px; margin-bottom: 12px; }
.empty-state h3 { color: #31514b; font-family: Georgia, serif; font-size: 22px; font-weight: 500; margin-bottom: 6px; }
.empty-state p { font-size: 14px; margin-bottom: 0; }
@media (max-width: 760px) { .page-shell { padding: 28px 16px 48px; } .intro { align-items: start; flex-direction: column; } .summary { min-width: 120px; } .workspace { grid-template-columns: 1fr; } .appointment-form, .appointment-list { padding: 20px 16px; } }
</style>
