<script setup>
import { createClient } from "@supabase/supabase-js";
import { onMounted, ref } from "vue";
import { Chart, registerables } from "chart.js";

// Enregistrement des composants nécessaires de Chart.js
Chart.register(...registerables);

// Configuration de Supabase à partir des variables d'environnement
const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

const banDates = ref([]);
const banCounts = ref([]);
let allData = [];
let chartInstance = null;
const availableYears = ref([]);

// Fonction pour récupérer les données des bans
const fetchBans = async () => {
  try {
    const { data, error } = await supabase
      .from("profil")
      .select("ban, ban_date")
      .filter("ban", "eq", true);

    if (error) {
      throw new Error(
        "Erreur lors de la récupération des données: " + error.message
      );
    }

    if (!data) {
      console.warn("Aucune donnée reçue de la base de données.");
      return;
    }

    allData = data;
    extractAvailableYears();
    filterDataByYear();
  } catch (err) {
    console.error(
      "Erreur réseau ou autre problème lors de la récupération des données",
      err
    );
  }
};

// Extraire les années disponibles
const extractAvailableYears = () => {
  const years = new Set();
  allData.forEach((record) => {
    if (record.ban_date) {
      const yearOnly = new Date(record.ban_date).getFullYear();
      years.add(yearOnly);
    }
  });
  availableYears.value = Array.from(years).sort((a, b) => a - b);
};

// Filtrer les données par année
const filterDataByYear = (year = null) => {
  const dates = [];
  const counts = [];
  const dateMap = {};

  allData.forEach((record) => {
    if (record.ban === true) {
      const dateOnly = new Date(record.ban_date).toISOString().split("T")[0];
      const yearOnly = new Date(record.ban_date).getFullYear();
      if (year === null || yearOnly === year) {
        if (!dateMap[dateOnly]) {
          dateMap[dateOnly] = 1;
        } else {
          dateMap[dateOnly]++;
        }
      }
    }
  });

  // Trier les dates du plus vieux au plus récent
  const sortedDates = Object.keys(dateMap).sort(
    (a, b) => new Date(a) - new Date(b)
  );

  sortedDates.forEach((date) => {
    dates.push(date);
    counts.push(dateMap[date]);
  });

  banDates.value = dates;
  banCounts.value = counts;
  if (chartInstance) {
    chartInstance.destroy();
  }
  createChart();
};

// Création du graphique
const createChart = () => {
  const canvasElement = document.getElementById("banChart");
  if (!canvasElement) {
    console.error("Canvas element 'banChart' not found.");
    return;
  }
  const ctx = canvasElement.getContext("2d");
  if (!ctx) {
    console.error("Unable to get 2D context for the canvas.");
    return;
  }

  chartInstance = new Chart(ctx, {
    type: "bar",
    data: {
      labels: banDates.value,
      datasets: [
        {
          data: banCounts.value,
          backgroundColor: banCounts.value.map(
            (count) =>
              `rgba(${Math.min(255, count * 20)}, ${
                100 - Math.min(50, count * 5)
              }, ${255 - Math.min(100, count * 10)}, 0.9)`
          ),
          borderColor: banCounts.value.map(
            (count) =>
              `rgba(${Math.min(255, count * 20)}, ${
                100 - Math.min(50, count * 5)
              }, ${255 - Math.min(100, count * 10)}, 1)`
          ),
          borderWidth: 1,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      scales: {
        y: {
          beginAtZero: true,
          grid: {
            color: "rgba(255, 255, 255, 0.1)",
          },
          ticks: {
            color: "#ffffff",
          },
        },
        x: {
          grid: {
            color: "rgba(255, 255, 255, 0.1)",
          },
          ticks: {
            color: "#ffffff",
          },
        },
      },
      plugins: {
        legend: {
          display: false,
        },
      },
    },
  });
};

onMounted(async () => {
  await fetchBans();
});
</script>

<template>
  <div class="bg-gray-900 text-white min-h-screen">
    <div class="navbar bg-base-100">
      <div class="flex-1">
        <a class="btn btn-ghost normal-case text-xl">Steam Ban Tracker Web</a>
      </div>
    </div>
    <div class="min-h-screen flex flex-col items-center justify-center">
      <div class="mb-6 flex space-x-4">
        <button
          @click="filterDataByYear(null)"
          class="btn btn-outline btn-secondary"
        >
          Tous les années
        </button>
        <button
          v-for="year in availableYears"
          :key="year"
          @click="filterDataByYear(year)"
          class="btn btn-outline btn-secondary"
        >
          {{ year }}
        </button>
      </div>
      <div class="w-full px-6">
        <canvas id="banChart" height="600"></canvas>
      </div>
    </div>
  </div>
  <footer class="footer footer-center bg-base-100 text-white p-4">
    <aside>
      <p>Made by Dr. Agorille</p>
    </aside>
  </footer>
</template>
