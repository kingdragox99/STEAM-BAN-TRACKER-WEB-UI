<script setup>
import { SpeedInsights } from "@vercel/speed-insights/vue";
import { createClient } from "@supabase/supabase-js";
import { onMounted, ref } from "vue";
import { Chart, registerables } from "chart.js";

// Registering necessary components of Chart.js
Chart.register(...registerables);

// Supabase configuration from environment variables
const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

const banDates = ref([]);
const banCounts = ref([]);
let allData = [];
let chartInstance = null;
const availableYears = ref([]);
const totalProfiles = ref(0);
const totalBanned = ref(0);
const selectedYear = ref(null);
const totalBannedInYear = ref(0);

// Function to fetch ban data
const fetchBans = async () => {
  try {
    let allResults = [];
    let start = 0;
    let limit = 1000;
    let fetchMore = true;

    while (fetchMore) {
      const { data, error } = await supabase
        .from("profil")
        .select("ban, ban_date")
        .filter("ban", "eq", true)
        .range(start, start + limit - 1);

      if (error) {
        throw new Error("Error fetching data: " + error.message);
      }

      if (data.length > 0) {
        allResults = allResults.concat(data);
        start += limit;
      } else {
        fetchMore = false;
      }
    }

    allData = allResults;
    extractAvailableYears();
    filterDataByYear();
  } catch (err) {
    console.error("Network error or other issue fetching data", err);
  }
};

// Function to fetch summary data
const fetchSummaryData = async () => {
  try {
    const { count, error } = await supabase
      .from("profil")
      .select("id", { count: "exact" });

    if (error) {
      throw new Error("Error fetching total profiles: " + error.message);
    }

    totalProfiles.value = count;

    const { count: bannedCount, error: bannedError } = await supabase
      .from("profil")
      .select("id", { count: "exact" })
      .eq("ban", true);

    if (bannedError) {
      throw new Error(
        "Error fetching banned users count: " + bannedError.message
      );
    }

    totalBanned.value = bannedCount;
  } catch (err) {
    console.error("Network error or other issue fetching summary data", err);
  }
};

// Extract available years
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

// Filter data by year
const filterDataByYear = (year = null) => {
  const dates = [];
  const counts = [];
  const dateMap = {};
  selectedYear.value = year;

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

  // Sort dates from oldest to newest
  const sortedDates = Object.keys(dateMap).sort(
    (a, b) => new Date(a) - new Date(b)
  );

  sortedDates.forEach((date) => {
    dates.push(date);
    counts.push(dateMap[date]);
  });

  banDates.value = dates;
  banCounts.value = counts;
  totalBannedInYear.value = counts.reduce((a, b) => a + b, 0);

  if (chartInstance) {
    // Update chart instead of destroying it for better performance
    chartInstance.data.labels = banDates.value;
    chartInstance.data.datasets[0].data = banCounts.value;
    chartInstance.data.datasets[0].backgroundColor = banCounts.value.map(
      (count) =>
        `rgba(${Math.min(255, count * 20)}, ${100 - Math.min(50, count * 5)}, ${
          255 - Math.min(100, count * 10)
        }, 0.9)`
    );
    chartInstance.update();
  } else {
    nextTick(() => {
      createChart();
    });
  }
};

// Creating the chart
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

const loading = ref(true);

import { nextTick } from "vue";

onMounted(async () => {
  await fetchBans();
  await fetchSummaryData();
  loading.value = false;
  await nextTick();
  createChart();
});
</script>

<template>
  <div v-if="loading" class="flex items-center justify-center min-h-screen">
    <div class="loading loading-dots loading-lg text-primary"></div>
  </div>
  <div v-else class="bg-gray-900 text-white mb-6">
    <div class="flex flex-col items-center justify-center">
      <div class="m-6 flex space-x-4">
        <button
          @click="filterDataByYear(null)"
          class="btn btn-outline btn-secondary"
        >
          All years
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
    <div class="flex items-center justify-center">
      <div class="stats shadow mt-6">
        <div class="stat">
          <div class="stat-title">Total profiles tracked</div>
          <div class="stat-value">{{ totalProfiles }}</div>
        </div>
        <div class="stat">
          <div class="stat-title">Total banned users</div>
          <div class="stat-value">{{ totalBanned }}</div>
        </div>
        <div class="stat">
          <div class="stat-title">Banned in {{ selectedYear ?? "total" }}</div>
          <div class="stat-value">{{ totalBannedInYear }}</div>
        </div>
      </div>
    </div>
  </div>
</template>
