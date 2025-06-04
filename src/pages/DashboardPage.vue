<template>
  <q-page class="dashboard">
    <q-layout view="hHh lpR fFf">
      <!-- Верхняя панель -->
      <q-header elevated class="bg-primary text-white">
        <q-toolbar>
          <q-toolbar-title>
            Учебный портал
          </q-toolbar-title>
          <q-btn flat round dense icon="person" class="q-ml-sm">
            <q-menu>
              <q-list style="min-width: 100px">
                <q-item clickable v-close-popup @click="logout">
                  <q-item-section>Выйти</q-item-section>
                </q-item>
              </q-list>
            </q-menu>
          </q-btn>
        </q-toolbar>
      </q-header>

      <!-- Боковое меню -->
      <q-drawer
        v-model="leftDrawerOpen"
        show-if-above
        bordered
        class="bg-grey-1"
      >
        <q-list>
          <q-item-label header>Меню</q-item-label>
          <q-item clickable v-ripple to="/dashboard">
            <q-item-section avatar>
              <q-icon name="dashboard" />
            </q-item-section>
            <q-item-section>Главная</q-item-section>
          </q-item>

          <q-item clickable v-ripple to="/courses">
            <q-item-section avatar>
              <q-icon name="school" />
            </q-item-section>
            <q-item-section>Курсы</q-item-section>
          </q-item>

          <q-item clickable v-ripple to="/schedule">
            <q-item-section avatar>
              <q-icon name="event" />
            </q-item-section>
            <q-item-section>Расписание</q-item-section>
          </q-item>

          <q-item clickable v-ripple to="/grades">
            <q-item-section avatar>
              <q-icon name="grade" />
            </q-item-section>
            <q-item-section>Оценки</q-item-section>
          </q-item>
        </q-list>
      </q-drawer>

      <!-- Основной контент -->
      <q-page-container>
        <q-page padding>
          <div class="row q-col-gutter-md">
            <!-- Карточка с приветствием -->
            <div class="col-12">
              <q-card class="welcome-card">
                <q-card-section>
                  <div class="text-h5">Добро пожаловать!</div>
                  <div class="text-subtitle2">Учебный портал</div>
                </q-card-section>
              </q-card>
            </div>

            <!-- Карточки с информацией -->
            <div class="col-12 col-md-4">
              <q-card>
                <q-card-section>
                  <div class="text-h6">Активные курсы</div>
                  <div class="text-h4 q-mt-sm">3</div>
                </q-card-section>
              </q-card>
            </div>

            <div class="col-12 col-md-4">
              <q-card>
                <q-card-section>
                  <div class="text-h6">Ближайшие занятия</div>
                  <div class="text-h4 q-mt-sm">2</div>
                </q-card-section>
              </q-card>
            </div>

            <div class="col-12 col-md-4">
              <q-card>
                <q-card-section>
                  <div class="text-h6">Средний балл</div>
                  <div class="text-h4 q-mt-sm">4.5</div>
                </q-card-section>
              </q-card>
            </div>
          </div>
        </q-page>
      </q-page-container>
    </q-layout>
  </q-page>
</template>

<script>
export default {
  name: 'DashboardPage',
  data() {
    return {
      leftDrawerOpen: true
    }
  },
  methods: {
    logout() {
      // Здесь будет логика выхода
      window.location.href = '/login'
    }
  }
}
</script>

<style scoped>
.dashboard {
  min-height: 100vh;
}
.welcome-card {
  background: linear-gradient(45deg, #1976D2, #2196F3);
  color: white;
}
</style> 