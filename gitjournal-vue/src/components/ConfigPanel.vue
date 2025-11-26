<script setup>
defineProps({
  config: Object,
  loading: Boolean,
  saveStatus: String,
  hasData: Boolean,
  currentTheme: String,
});

defineEmits(["generate", "export", "update-theme"]);
</script>

<template>
  <header class="no-print">
    <div class="title-container">
      <h1>
        <img src="/icon.png" alt="Logo" class="logo-img glow-img" />
        <span class="neon-text">GitJournal</span>
      </h1>
    </div>

    <div class="config-box">
      <div class="form-row">
        <input
          v-model="config.token"
          type="password"
          placeholder="Token GitHub"
          class="std-input"
        />
        <input
          v-model="config.author"
          type="text"
          placeholder="Filtre Auteur"
          class="std-input"
        />
      </div>
      <div class="form-row">
        <input
          v-model="config.owner"
          type="text"
          placeholder="User/Org"
          class="std-input"
        />
        <input
          v-model="config.repo"
          type="text"
          placeholder="Repo"
          class="std-input"
        />
        <button @click="$emit('generate')" :disabled="loading" class="btn-gen">
          {{ loading ? "..." : "Générer" }}
        </button>
      </div>
      <div class="tips">
        <small
          >Status:
          <span :class="{ ok: saveStatus.includes('✓') }">{{
            saveStatus || "Prêt"
          }}</span></small
        >
      </div>
      <button v-if="hasData" @click="$emit('export')" class="btn-pdf">
        Exporter PDF
      </button>
    </div>
  </header>
</template>

<style scoped>
/* --- THÈME NÉON TECH (Inchangé) --- */
h1 {
  font-family: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
    sans-serif;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 3.5rem;
  font-weight: 800;
  margin: 10px 0 25px 0;
  text-transform: uppercase;
  letter-spacing: -1px;
}

.neon-text {
  color: #42b883;
  text-shadow: 0 0 2px #fff, 0 0 10px #42b883, 0 0 20px rgba(66, 184, 131, 0.5),
    0 0 40px rgba(66, 184, 131, 0.3);
}

.logo-img {
  height: 60px;
  width: auto;
  margin-right: 20px;
  filter: drop-shadow(0 0 8px rgba(66, 184, 131, 0.8));
}

.config-box {
  background: white;
  padding: 25px;
  border-radius: 16px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.05);
  margin-bottom: 30px;
  border: 1px solid #f0f0f0;
}

/* --- FIX LAYOUT --- */
.form-row {
  display: flex;
  gap: 12px;
  margin-bottom: 15px;
  /* FIX 1 : Permet aux éléments de passer à la ligne si c'est trop étroit */
  flex-wrap: wrap;
}

.std-input {
  flex: 1; /* Les inputs prennent toute la place dispo */
  min-width: 120px; /* FIX 2 : Empêche les inputs de devenir illisibles */
  padding: 12px;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
  transition: all 0.2s;
}

.std-input:focus {
  outline: none;
  border-color: #42b883;
  box-shadow: 0 0 0 3px rgba(66, 184, 131, 0.1);
}

.btn-gen {
  background: #42b883;
  color: white;
  border: none;
  padding: 0 25px;
  border-radius: 8px;
  cursor: pointer;
  font-weight: bold;
  font-size: 1rem;
  white-space: nowrap; /* FIX 3 : Empêche le texte "Générer" de se couper */
  transition: all 0.2s;
  box-shadow: 0 4px 6px rgba(66, 184, 131, 0.2);
  /* FIX 4 : Hauteur minimale pour s'aligner avec les inputs */
  height: 45px;
  align-self: flex-start; /* Évite l'étirement bizarre si la ligne grandit */
}

.btn-gen:hover {
  background: #3aa876;
  box-shadow: 0 4px 12px rgba(66, 184, 131, 0.4);
  transform: translateY(-1px);
}

.btn-gen:disabled {
  background: #a0aec0;
  cursor: not-allowed;
  box-shadow: none;
}

.btn-pdf {
  width: 100%;
  background: #2c3e50;
  color: white;
  border: none;
  padding: 12px;
  border-radius: 8px;
  cursor: pointer;
  margin-top: 15px;
  font-weight: bold;
  font-size: 1rem;
  transition: background 0.2s;
}

.btn-pdf:hover {
  background: #1a252f;
}

.tips {
  text-align: center;
  color: #718096;
  margin-top: 5px;
  font-size: 0.85em;
  height: 20px;
}

.ok {
  color: #42b883;
  font-weight: bold;
  text-shadow: 0 0 5px rgba(66, 184, 131, 0.5);
}

/* FIX 5 : Media Query pour forcer l'empilement sur mobile/petits écrans */
@media (max-width: 600px) {
  .form-row {
    flex-direction: column; /* Empile verticalement */
  }
  .btn-gen {
    width: 100%; /* Le bouton prend toute la largeur */
  }
}
</style>
