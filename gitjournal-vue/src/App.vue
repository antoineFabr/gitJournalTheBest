<script setup>
import { ref, onMounted } from "vue";
import { fr } from "date-fns/locale"; // Attention à l'import format ici
import { format } from "date-fns"; // Attention à l'import format ici
import { differenceInMinutes, parseISO } from "date-fns";
import html2pdf from "html2pdf.js";

// Imports des composants et utils
import ConfigPanel from "./components/ConfigPanel.vue";
import DayCard from "./components/DayCard.vue";
import { extractTags } from "./utils/gitParser";

const API_URL = "http://localhost:3000/edits";
const config = ref({ token: "", owner: "", repo: "", author: "" });

// État
const savedEdits = ref({});
const journalData = ref({});
const loading = ref(false);
const error = ref(null);
const saveStatus = ref("");

onMounted(async () => {
  config.value.token = localStorage.getItem("gj_token") || "";
  config.value.owner = localStorage.getItem("gj_owner") || "";
  config.value.repo = localStorage.getItem("gj_repo") || "";
  config.value.author = localStorage.getItem("gj_author") || "";
  await loadEditsFromServer();
});

const loadEditsFromServer = async () => {
  try {
    const res = await fetch(API_URL);
    if (res.ok) savedEdits.value = await res.json();
  } catch (e) {
    console.error("Serveur backend injoignable");
    error.value = "Info: Serveur de sauvegarde non détecté.";
  }
};

const saveConfig = () => {
  localStorage.setItem("gj_token", config.value.token);
  localStorage.setItem("gj_owner", config.value.owner);
  localStorage.setItem("gj_repo", config.value.repo);
  localStorage.setItem("gj_author", config.value.author);
};

let timeoutId = null;
const handleCommitUpdate = (commit) => {
  // Mise à jour de l'état local pour la sauvegarde
  savedEdits.value[commit.id] = {
    message: commit.message,
    duration: commit.duration,
    status: commit.status,
  };

  // Debounce sauvegarde
  saveStatus.value = "...";
  clearTimeout(timeoutId);
  timeoutId = setTimeout(async () => {
    try {
      await fetch(API_URL, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(savedEdits.value),
      });
      saveStatus.value = "Sauvegardé ✓";
      setTimeout(() => {
        saveStatus.value = "";
      }, 2000);
    } catch (e) {
      saveStatus.value = "Erreur Sauvegarde ❌";
    }
  }, 1000);
};

const fetchCommits = async () => {
  saveConfig();
  await loadEditsFromServer();

  if (!config.value.owner || !config.value.repo) {
    error.value = "Veuillez remplir le propriétaire et le dépôt.";
    return;
  }

  loading.value = true;
  error.value = null;
  journalData.value = {};

  try {
    let url = `https://api.github.com/repos/${config.value.owner}/${config.value.repo}/commits?per_page=100`;
    if (config.value.author) url += `&author=${config.value.author}`;
    const headers = { Accept: "application/vnd.github.v3+json" };
    if (config.value.token)
      headers["Authorization"] = `token ${config.value.token}`;

    const res = await fetch(url, { headers });
    if (!res.ok) throw new Error(`Erreur API: ${res.status}`);

    let rawCommits = await res.json();
    rawCommits.reverse();

    const processedCommits = rawCommits.map((commit, index) => {
      const currentDate = parseISO(commit.commit.author.date);
      const sha = commit.sha;

      // Utilisation du parser importé
      const { fullCleanMsg, manualDuration, status } = extractTags(
        commit.commit.message
      );
      const titleOnly = fullCleanMsg.split("\n")[0].trim();

      // Calcul Auto
      let autoDuration = 0;
      if (index > 0) {
        const prevDate = parseISO(rawCommits[index - 1].commit.author.date);
        const diff = differenceInMinutes(currentDate, prevDate);
        if (diff < 180) autoDuration = diff;
      }

      const saved = savedEdits.value[sha];

      let finalDuration = autoDuration;
      if (manualDuration !== null) finalDuration = manualDuration;
      if (saved && saved.duration !== undefined) finalDuration = saved.duration;

      const finalMessage = saved?.message || titleOnly;

      // --- MODIFICATION : "DONE" par défaut ---
      const finalStatus = saved?.status || status || "DONE";

      return {
        id: sha,
        message: finalMessage,
        date: currentDate,
        timeStr: format(currentDate, "HH:mm"),
        dateKey: format(currentDate, "EEEE d MMMM yyyy", { locale: fr }),
        duration: finalDuration,
        status: finalStatus,
        url: commit.html_url,
      };
    });

    processedCommits.reverse();

    const grouped = {};
    processedCommits.forEach((c) => {
      if (!grouped[c.dateKey])
        grouped[c.dateKey] = { commits: [], totalMinutes: 0 };
      grouped[c.dateKey].commits.push(c);
      grouped[c.dateKey].totalMinutes += c.duration;
    });

    journalData.value = grouped;
  } catch (e) {
    error.value = e.message;
  } finally {
    loading.value = false;
  }
};

const exportPDF = () => {
  const element = document.getElementById("printable-area");
  const opt = {
    margin: 10,
    filename: `Journal-${config.value.repo}.pdf`,
    image: { type: "jpeg", quality: 0.98 },
    html2canvas: { scale: 2, useCORS: true },
    jsPDF: { unit: "mm", format: "a4", orientation: "portrait" },
  };
  html2pdf().set(opt).from(element).save();
};
</script>

<template>
  <div class="app-container">
    <ConfigPanel
      :config="config"
      :loading="loading"
      :saveStatus="saveStatus"
      :hasData="Object.keys(journalData).length > 0"
      @generate="fetchCommits"
      @export="exportPDF"
    />

    <div v-if="error" class="error">{{ error }}</div>

    <main id="printable-area">
      <div v-if="Object.keys(journalData).length" class="report-header">
        <h2>Journal : {{ config.repo }}</h2>
        <p>Généré le {{ new Date().toLocaleDateString() }}</p>
      </div>

      <DayCard
        v-for="(data, date) in journalData"
        :key="date"
        :date="date"
        :data="data"
        @commit-updated="handleCommitUpdate"
      />
    </main>
  </div>
</template>

<style scoped>
.app-container {
  font-family: "Inter", sans-serif;
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  color: #2c3e50;
}
.error {
  color: #e74c3c;
  font-weight: bold;
  text-align: center;
  margin-bottom: 20px;
}
.report-header {
  text-align: center;
  margin-bottom: 30px;
  display: none;
}
#printable-area .report-header {
  display: block;
}
</style>
