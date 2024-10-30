<script setup>
import { createClient } from "@supabase/supabase-js";
import { onMounted, ref, onBeforeUnmount, nextTick, watch } from "vue";
import { Chart, registerables } from "chart.js";

Chart.register(...registerables);

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

const banDates = ref([]);
const banCounts = ref({ vacBan: [], gameBan: [], unban: [], tradeBan: [] });
const totalBansForSelectedYear = ref(0);
const vacBanCount = ref(0);
const gameBanCount = ref(0);
const unbanCount = ref(0);
const tradeBanCount = ref(0);
let allData = [];
let chartInstance = null;
const availableYears = ref([]);
const selectedYear = ref(null);
const isLoading = ref(true);
const showDetails = ref(true);

const fetchBans = async () => {
  try {
    let allResults = [];
    let start = 0;
    const limit = 1000;
    let fetchMore = true;

    while (fetchMore) {
      const { data, error } = await supabase
        .from("profil")
        .select("ban, ban_date, ban_type")
        .eq("ban", true)
        .range(start, start + limit - 1);

      if (error) {
        console.error("Error fetching data: ", error);
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
  } finally {
    isLoading.value = false;
    nextTick(() => {
      createChart();
    });
  }
};

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

const filterDataByYear = (year = null) => {
  const dateMap = {};
  selectedYear.value = year;
  totalBansForSelectedYear.value = 0;
  vacBanCount.value = 0;
  gameBanCount.value = 0;
  unbanCount.value = 0;
  tradeBanCount.value = 0;

  allData.forEach((record) => {
    if (record.ban === true) {
      const dateOnly = new Date(record.ban_date).toISOString().split("T")[0];
      const yearOnly = new Date(record.ban_date).getFullYear();
      if (year === null || yearOnly === year) {
        if (!dateMap[dateOnly]) {
          dateMap[dateOnly] = { vac: 0, game: 0, unban: 0, trade: 0 };
        }
        const banType = record.ban_type.trim().toLowerCase();
        if (banType === "vac ban") {
          dateMap[dateOnly].vac++;
          vacBanCount.value++;
        } else if (banType === "game ban") {
          dateMap[dateOnly].game++;
          gameBanCount.value++;
        } else if (banType === "unban") {
          dateMap[dateOnly].unban++;
          unbanCount.value++;
        } else if (banType === "trade ban") {
          dateMap[dateOnly].trade++;
          tradeBanCount.value++;
        }
        totalBansForSelectedYear.value++;
      }
    }
  });

  const sortedDates = Object.keys(dateMap).sort(
    (a, b) => new Date(a) - new Date(b)
  );
  banDates.value = sortedDates;
  banCounts.value = {
    vacBan: sortedDates.map((date) => dateMap[date].vac),
    gameBan: sortedDates.map((date) => dateMap[date].game),
    unban: sortedDates.map((date) => dateMap[date].unban),
    tradeBan: sortedDates.map((date) => dateMap[date].trade),
  };

  updateChart();
};

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
          label: "VAC Ban",
          data: banCounts.value.vacBan,
          backgroundColor: "rgba(233, 46, 144, 1)",
        },
        {
          label: "Game Ban",
          data: banCounts.value.gameBan,
          backgroundColor: "rgba(21, 91, 224, 1)",
        },
        {
          label: "Unban",
          data: banCounts.value.unban,
          backgroundColor: "rgba(46, 233, 135, 1)",
        },
        {
          label: "Trade Ban",
          data: banCounts.value.tradeBan,
          backgroundColor: "rgba(255, 165, 0, 1)",
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
            color: getGridColor(),
          },
          ticks: {
            color: getTextColor(),
          },
        },
        x: {
          grid: {
            color: getGridColor(),
          },
          ticks: {
            color: getTextColor(),
          },
        },
      },
      plugins: {
        legend: {
          display: true,
          labels: {
            color: getTextColor(),
          },
        },
      },
    },
  });
};

const updateChart = () => {
  if (chartInstance) {
    chartInstance.data.labels = banDates.value;
    chartInstance.data.datasets[0].data = banCounts.value.vacBan;
    chartInstance.data.datasets[1].data = banCounts.value.gameBan;
    chartInstance.data.datasets[2].data = banCounts.value.unban;
    chartInstance.data.datasets[3].data = banCounts.value.tradeBan;
    updateChartColors();
    chartInstance.update();
  } else {
    nextTick(() => {
      createChart();
    });
  }
};

const getTextColor = () => {
  const theme =
    document.querySelector("html").getAttribute("data-theme") || "dark";
  return theme === "light" ? "#000000" : "#ffffff";
};

const getGridColor = () => {
  const theme =
    document.querySelector("html").getAttribute("data-theme") || "dark";
  return theme === "light" ? "rgba(0, 0, 0, 0.1)" : "rgba(255, 255, 255, 0.1)";
};

const updateChartColors = () => {
  if (chartInstance) {
    chartInstance.options.scales.x.ticks.color = getTextColor();
    chartInstance.options.scales.y.ticks.color = getTextColor();
    chartInstance.options.plugins.legend.labels.color = getTextColor();
    chartInstance.options.scales.x.grid.color = getGridColor();
    chartInstance.options.scales.y.grid.color = getGridColor();
  }
};

onMounted(() => {
  const handleThemeChange = () => {
    updateChartColors();
    if (chartInstance) chartInstance.update();
  };

  const observer = new MutationObserver(handleThemeChange);
  observer.observe(document.documentElement, {
    attributes: true,
    attributeFilter: ["data-theme"],
  });

  onBeforeUnmount(() => {
    observer.disconnect();
    if (chartInstance) chartInstance.destroy();
  });

  fetchBans();
});
</script>

<template>
  <div
    v-if="isLoading"
    class="flex items-center justify-center min-h-screen animate-pulse"
  >
    <div class="loading loading-dots text-primary w-24"></div>
  </div>
  <div v-else>
    <div class="flex flex-col items-center justify-center w-full">
      <div class="p-4 w-3/4">
        <div class="sm:hidden w-full text-center">
          <select
            class="select select-bordered w-full"
            @change="
              filterDataByYear(
                $event.target.value === 'null'
                  ? null
                  : parseInt($event.target.value)
              )
            "
          >
            <option value="null" :selected="selectedYear === null">
              All years
            </option>
            <option
              v-for="year in availableYears"
              :key="year"
              :value="year"
              :selected="selectedYear === year"
            >
              {{ year }}
            </option>
          </select>
        </div>
        <div class="hidden sm:flex flex-wrap justify-center gap-4">
          <button
            @click="filterDataByYear(null)"
            :class="[
              'btn',
              selectedYear === null ? 'btn-active' : 'btn-outline',
              'btn-secondary',
            ]"
          >
            All years
          </button>
          <button
            v-for="year in availableYears"
            :key="year"
            @click="filterDataByYear(year)"
            :class="[
              'btn',
              selectedYear === year ? 'btn-active' : 'btn-outline',
              'btn-secondary',
            ]"
          >
            {{ year }}
          </button>
        </div>
      </div>
      <button
        @click="showDetails = !showDetails"
        :class="[
          'btn sm:hidden mb-4',
          showDetails === true ? 'btn-active' : 'btn-outline',
          'btn-secondary',
        ]"
      >
        {{ showDetails ? "Hide Details" : "Show Details" }}
      </button>
      <div class="w-full px-4 sm:px-6 lg:px-8 relative">
        <canvas id="banChart" height="600"></canvas>
        <div
          v-show="showDetails"
          class="absolute top-14 right-3 text-lg bg-base-100 p-2 rounded-md shadow-md flex flex-col gap-1 sm:top-10 sm:right-10"
        >
          <p class="font-bold text-secondary">
            Total banned: {{ totalBansForSelectedYear }}
          </p>
          <p class="font-bold" style="color: rgba(233, 46, 144, 1)">
            VAC Ban: {{ vacBanCount }}
          </p>
          <p class="font-bold" style="color: rgba(21, 91, 224, 1)">
            Game Ban: {{ gameBanCount }}
          </p>
          <p class="font-bold" style="color: rgba(46, 233, 135, 1)">
            Unban: {{ unbanCount }}
          </p>
          <p class="font-bold" style="color: rgba(255, 165, 0, 1)">
            Trade Ban: {{ tradeBanCount }}
          </p>
        </div>
      </div>
    </div>
  </div>
</template>
