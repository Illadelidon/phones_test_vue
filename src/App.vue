<template>
  <div id="app">
    <ContactsTable />
  </div>
</template>

<script>
import ContactsTable from './components/MainPage.vue';



export default {
  name: 'App',
  components: {
    ContactsTable
  }
};
</script>

<style>
#app {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
}
</style>