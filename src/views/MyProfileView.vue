<template>
  <div class="profile-container">
    <h1>Welcome, {{ user ? user.name : '' }}</h1>

    <div v-if="error" class="error-message">{{ error }}</div>
    <div v-else-if="!user" class="loading">Loading...</div>

    <div v-if="user" class="user-details">
      <img :src="user.photo ? user.photo.startsWith('/api/photo/') ? `http://localhost:5001${user.photo}` : `http://localhost:5001/api/photo/${user.photo}` : '/default-profile.png'" alt="Profile Photo" class="user-photo" />
      <h2>{{ user.name }}</h2>
      <p><strong>Username:</strong> {{ user.username }}</p>
      <p><strong>Email:</strong> {{ user.email }}</p>
      <p><strong>Joined:</strong> {{ new Date(user.date_joined).toLocaleDateString() }}</p>

      <button @click="editUserProfile(user.id)" class="btn">Edit Profile</button>
      <button @click="createProfile" class="btn" style="margin-left: 10px;">Add Profile</button>
      <p v-if="warning" class="warning-message">{{ warning }}</p>
    </div>

    <h2>My Profiles</h2>
    <div v-if="profiles.length" class="profiles-grid">
      <div v-for="profile in profiles" :key="profile.id" class="profile-card">
        <img :src="profile.photo ? profile.photo : '/default-profile.png'" alt="Profile Photo" class="profile-photo" />
        <h3>{{ profile.name }}</h3>
        <p><strong>Parish:</strong> {{ profile.parish }}</p>
        <p><strong>Biography:</strong> {{ profile.biography }}</p>
        <button @click="viewProfile(profile.id)" class="action-button">View Profile</button>
        <button @click="findMatches(profile.id)" class="action-button">Match Me</button>
        <button @click="deleteProfile(profile.id)" class="delete-button" title="Delete Profile">
          🗑️
        </button>
        <button @click="editProfile(profile.id)" class="action-button" style="margin-left: 5px;">Edit</button>
      </div>
    </div>
    <p v-else>No profiles found.</p>

    <div v-if="matchingProfiles.length" class="match-report">
      <h2>Match Report</h2>
      <button @click="closeMatchReport" class="close-button">Close</button>
      <div class="match-grid">
        <div v-for="match in matchingProfiles" :key="match.profile_id" class="match-card">
          <img :src="match.photo ? `http://localhost:5001${match.photo}` : '/default-profile.png'" alt="Match Photo" class="match-photo" />
          <h3>{{ match.name }}</h3>
          <p><strong>Parish:</strong> {{ match.parish }}</p>
          <p><strong>Biography:</strong> {{ match.biography }}</p>
        </div>
      </div>
    </div>

    <h2>My Favorites</h2>
    <div v-if="favorites.length" class="favorites-container">
      <div v-for="fav in favorites" :key="fav.id" class="profile-card">
        <img :src="fav.photo ? `http://localhost:5001${fav.photo}` : '/default-profile.png'" alt="Favorite User Photo" class="profile-photo" />
        <h3>{{ fav.name }}</h3>
        <p><strong>Username:</strong> {{ fav.username }}</p>
        <p><strong>Email:</strong> {{ fav.email }}</p>
      </div>
    </div>
    <p v-else>No favorite users yet.</p>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import { useRouter } from "vue-router";
import { useAuthStore } from '../stores/auth';

const authStore = useAuthStore();
const token = authStore.token;

const router = useRouter();
const user = ref(null);
const profiles = ref([]);
const favorites = ref([]);
const error = ref("");
const warning = ref("");
const matchingProfiles = ref([]);

async function fetchData(url, errorMessage) {
  try {
    const res = await fetch(`http://localhost:5001${url}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${token}` },
      credentials: 'include'
    });
    if (!res.ok) {
      throw new Error(errorMessage);
    }
    return await res.json();
  } catch (err) {
    error.value = err.message;
  }
}

async function deleteProfileApi(profileId) {
  try {
    let csrfToken = null;
    const match = document.cookie.match(new RegExp('(^| )csrf_access_token=([^;]+)'));
    if (match) {
      csrfToken = match[2];
    } else {
      const resToken = await fetch('http://localhost:5001/api/v1/csrf-token', { credentials: 'include' });
      if (resToken.ok) {
        const data = await resToken.json();
        csrfToken = data.csrf_token;
      }
    }
    if (!csrfToken) {
      throw new Error("CSRF token not found");
    }
    const res = await fetch(`http://localhost:5001/api/profiles/${profileId}`, {
      method: "DELETE",
      headers: { 
        Authorization: `Bearer ${token}`,
        'X-CSRFToken': csrfToken,
        'Content-Type': 'application/json'
      },
      credentials: 'include'
    });
    if (!res.ok) {
      const errorText = await res.text();
      throw new Error("Failed to delete profile");
    }
    return true;
  } catch (err) {
    error.value = err.message;
    return false;
  }
}

async function loadData() {
  const userId = authStore.user && authStore.user.id ? authStore.user.id : null;

  if (!userId) {
    error.value = "User not logged in or user ID not available.";
    return;
  }

  user.value = await fetchData(`/api/users/${userId}`, "Failed to load user details.");
  profiles.value = await fetchData(`/api/profiles/user/${userId}`, "Failed to load profiles.");
  favorites.value = await fetchData(`/api/users/${userId}/favourites`, "Failed to load favorite users.");
}

onMounted(() => {
  loadData();
});

function viewProfile(profileId) {
  router.push(`/profiles/${profileId}`);
}

async function findMatches(profileId) {
  try {
    const response = await fetch(`http://localhost:5001/api/profiles/matches/${profileId}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${token}` },
      credentials: 'include'
    });
    if (!response.ok) {
      throw new Error(`Failed to fetch matches: ${response.statusText}`);
    }
    const data = await response.json();
    if (data.matching_profiles && data.matching_profiles.length) {
      matchingProfiles.value = data.matching_profiles;
    } else {
      alert("No matches found.");
      matchingProfiles.value = [];
    }
  } catch (error) {
    error.value = error.message;
  }
}

function closeMatchReport() {
  matchingProfiles.value = [];
}

function createProfile() {
  if (profiles.value.length >= 2) {
    warning.value = "You can only have up to 3 profiles.";
    return;
  }
  warning.value = "";
  router.push('/create');
}

function editProfile(profileId) {
  if (!profileId) {
    alert('No profile selected for editing.');
    return;
  }
  router.push(`/edit-profile/${profileId}`);
}

function editUserProfile(userId) {
  if (!userId) {
    alert('No user selected for editing.');
    return;
  }
  router.push(`/edit-user-profile/${userId}`);
}

async function deleteProfile(profileId) {
  if (confirm("Are you sure you want to delete this profile?")) {
    const success = await deleteProfileApi(profileId);
    if (success) {
      profiles.value = profiles.value.filter(p => p.id !== profileId);
    }
  }
}
</script>

<style scoped>
.profile-container {
  max-width: 700px;
  margin: 50px auto;
  padding: 20px;
  background-color: #fff9f9;
  border-radius: 15px;
  box-shadow: 0 4px 8px rgba(255, 105, 97, 0.2);
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  text-align: center;
  color: #333;
}

h1 {
  color: #ff6f61;
  margin-bottom: 20px;
}

.user-details {
  margin-bottom: 30px;
}

.user-photo {
  width: 120px;
  height: 120px;
  border-radius: 50%;
  object-fit: cover;
  margin-bottom: 15px;
  border: 3px solid #ff6f61;
}

.btn {
  background: linear-gradient(135deg, #ffb347 0%, #ff69b4 100%);
  color: white;
  padding: 10px 18px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: bold;
  font-size: 1rem;
  transition: background 0.3s ease;
  margin: 0 5px;
}

.btn:hover {
  background: linear-gradient(135deg, #ffcc70 0%, #ff85c1 100%);
}

.profiles-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 20px;
  margin-top: 20px;
}

.profile-card, .fav-card {
  padding: 15px;
  border-radius: 12px;
  background: white;
  box-shadow: 0 2px 6px rgba(255, 105, 97, 0.15);
  position: relative;
  text-align: left;
  color: #333;
}

.profile-card h3 {
  color: #ff6f61;
  margin-bottom: 10px;
}

.profile-photo {
  width: 140px;
  height: 140px;
  border-radius: 50%;
  object-fit: cover;
  margin-bottom: 10px;
  border: 2px solid #ff6f61;
}

.action-button {
  background: linear-gradient(135deg, #ffb347 0%, #ff69b4 100%);
  color: white;
  padding: 6px 12px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  margin-right: 8px;
  transition: background 0.3s ease;
  font-size: 0.9rem;
}

.action-button:hover {
  background: linear-gradient(135deg, #ffcc70 0%, #ff85c1 100%);
}

.delete-button {
  background: transparent;
  border: none;
  color: #dc3545;
  font-size: 20px;
  cursor: pointer;
  position: absolute;
  top: 10px;
  right: 10px;
}

.delete-button:hover {
  color: #a71d2a;
}

.match-report {
  background-color: #fff9f9;
  border-radius: 15px;
  padding: 20px;
  margin-top: 30px;
  color: #333;
}

.match-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 15px;
}

.match-card {
  background-color: #fff9f9;
  border-radius: 10px;
  padding: 15px;
  text-align: center;
  color: #333;
}

.match-photo {
  width: 50px;
  height: 50px;
  border-radius: 50%;
  object-fit: cover;
  margin-bottom: 10px;
  border: 2px solid #ff6f61;
}

.close-button {
  background: linear-gradient(135deg, #ffb347 0%, #ff69b4 100%);
  color: white;
  border: none;
  padding: 8px 12px;
  border-radius: 8px;
  cursor: pointer;
  float: right;
  transition: background 0.3s ease;
}

.close-button:hover {
  background: linear-gradient(135deg, #ffcc70 0%, #ff85c1 100%);
}

.warning-message {
  color: #dc3545;
  margin-top: 10px;
  font-weight: bold;
}
</style>
