<template>
  <div class="home-page">
    <h1>Welcome to the Profile Matching App</h1>
    <p>Please log in or register to continue.</p>
    <div class="buttons">
      <router-link to="/login" class="btn">Login</router-link>
      <router-link to="/register" class="btn">Register</router-link>
    </div>
  </div>
</template>

<script setup>
import { useRoute } from "vue-router";
import LoginForm from '../components/LoginForm.vue';
import SignUpForm from '../components/SignUpForm.vue';

const route = useRoute();
</script>

<style scoped>
.home-page {
  text-align: center;
  margin-top: 100px;
}

.buttons {
  margin-top: 20px;
}

.btn {
  display: inline-block;
  margin: 0 10px;
  padding: 10px 20px;
  background-color: pink;
  color: white;
  text-decoration: none;
  border-radius: 5px;
  font-weight: bold;
}

.btn:hover {
  background-color: deeppink;
}
</style>
