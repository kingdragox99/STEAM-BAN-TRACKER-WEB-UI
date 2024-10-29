<script setup>
import { createClient } from "@supabase/supabase-js";
import { onMounted, ref, onBeforeUnmount, nextTick } from "vue";
import { Chart, registerables } from "chart.js";

Chart.register(...registerables);

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

const banDates = ref([]);
const vacBanCounts = ref([]);
const gameBanCounts = ref([]);
const unbanCounts = ref([]);
let allData = [];
let chartInstance = null;
const availableYears = ref([]);
const totalProfiles = ref(0);
const totalBanned = ref(0);
const selectedYear = ref(null);
const totalBannedInYear = ref(0);
const chartLoading = ref(true);
const banTypeCounts = ref({ vacBan: 0, gameBan: 0, tradeBan: 0, unban: 0 });

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
    calculateBanTypeCounts();
    extractAvailableYears();
    filterDataByYear();
  } catch (err) {
    console.error("Network error or other issue fetching data", err);
  }
};

const fetchSummaryData = async () => {
  try {
    const { count, error } = await supabase
      .from("profil")
      .select("id", { count: "exact" });

    if (error) {
      console.error("Error fetching total profiles: ", error);
      throw new Error("Error fetching total profiles: " + error.message);
    }

    totalProfiles.value = count;

    const { count: bannedCount, error: bannedError } = await supabase
      .from("profil")
      .select("id", { count: "exact" })
      .eq("ban", true);

    if (bannedError) {
      console.error("Error fetching banned users count: ", bannedError);
      throw new Error(
        "Error fetching banned users count: " + bannedError.message
      );
    }

    totalBanned.value = bannedCount;
  } catch (err) {
    console.error("Network error or other issue fetching summary data", err);
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

const calculateBanTypeCounts = () => {
  const counts = { vacBan: 0, gameBan: 0, tradeBan: 0, unban: 0 };
  allData.forEach((record) => {
    if (record.ban_type) {
      const normalizedType = record.ban_type.trim().toLowerCase();
      if (normalizedType === "vac ban") {
        counts.vacBan++;
      } else if (normalizedType === "game ban") {
        counts.gameBan++;
      } else if (normalizedType === "trade ban") {
        counts.tradeBan++;
      } else if (normalizedType === "unban") {
        counts.unban++;
      }
    }
  });
  banTypeCounts.value = counts;
};

const filterDataByYear = (year = null) => {
  const dates = [];
  const vacCounts = [];
  const gameCounts = [];
  const unbanCountsArr = [];
  const dateMap = {};
  selectedYear.value = year;

  allData.forEach((record) => {
    if (record.ban === true) {
      const dateOnly = new Date(record.ban_date).toISOString().split("T")[0];
      const yearOnly = new Date(record.ban_date).getFullYear();
      if (year === null || yearOnly === year) {
        if (!dateMap[dateOnly]) {
          dateMap[dateOnly] = { vac: 0, game: 0, unban: 0 };
        }
        if (record.ban_type.trim().toLowerCase() === "vac ban") {
          dateMap[dateOnly].vac++;
        } else if (record.ban_type.trim().toLowerCase() === "game ban") {
          dateMap[dateOnly].game++;
        } else if (record.ban_type.trim().toLowerCase() === "unban") {
          dateMap[dateOnly].unban++;
        }
      }
    }
  });

  const sortedDates = Object.keys(dateMap).sort(
    (a, b) => new Date(a) - new Date(b)
  );

  sortedDates.forEach((date) => {
    dates.push(date);
    vacCounts.push(dateMap[date].vac);
    gameCounts.push(dateMap[date].game);
    unbanCountsArr.push(dateMap[date].unban);
  });

  banDates.value = dates;
  vacBanCounts.value = vacCounts;
  gameBanCounts.value = gameCounts;
  unbanCounts.value = unbanCountsArr;
  totalBannedInYear.value =
    vacCounts.reduce((a, b) => a + b, 0) +
    gameCounts.reduce((a, b) => a + b, 0) +
    unbanCountsArr.reduce((a, b) => a + b, 0);

  if (chartInstance) {
    chartInstance.data.labels = banDates.value;
    chartInstance.data.datasets[0].data = vacBanCounts.value;
    chartInstance.data.datasets[1].data = gameBanCounts.value;
    chartInstance.data.datasets[2].data = unbanCounts.value;
    updateChartColors();
    chartInstance.update();
  } else {
    nextTick(() => {
      createChart();
    });
  }
  chartLoading.value = false;
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

  if (chartInstance) {
    chartInstance.destroy();
  }

  chartInstance = new Chart(ctx, {
    type: "bar",
    data: {
      labels: banDates.value,
      datasets: [
        {
          label: "VAC Ban",
          data: vacBanCounts.value,
          backgroundColor: "rgba(233, 46, 144, 1)",
        },
        {
          label: "Game Ban",
          data: gameBanCounts.value,
          backgroundColor: "rgba(21, 91, 224, 1)",
        },
        {
          label: "Unban",
          data: unbanCounts.value,
          backgroundColor: "rgba(46, 233, 135, 1)",
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

const getTextColor = () => {
  if (typeof window !== "undefined") {
    const theme =
      document.querySelector("html").getAttribute("data-theme") || "dark";
    return theme === "light" ? "#000000" : "#ffffff";
  }
  return "#000000";
};

const getGridColor = () => {
  if (typeof window !== "undefined") {
    const theme =
      document.querySelector("html").getAttribute("data-theme") || "dark";
    return theme === "light"
      ? "rgba(0, 0, 0, 0.1)"
      : "rgba(255, 255, 255, 0.1)";
  }
  return "rgba(0, 0, 0, 0.1)";
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
    const newTheme = document.documentElement.getAttribute("data-theme");
    updateChartColors();
    if (chartInstance) {
      chartInstance.update();
    }
  };

  const observer = new MutationObserver(handleThemeChange);
  observer.observe(document.documentElement, {
    attributes: true,
    attributeFilter: ["data-theme"],
  });

  onBeforeUnmount(() => {
    observer.disconnect();
  });
});

const loading = ref(true);

onMounted(async () => {
  await fetchBans();
  await fetchSummaryData();
  loading.value = false;
  await nextTick();
  createChart();
});
</script>

<template>
  <div v-show="loading" class="flex items-center justify-center min-h-screen">
    <div class="loading loading-dots text-primary w-24"></div>
  </div>
  <div v-show="!loading">
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
      <div class="w-full px-4 sm:px-6 lg:px-8">
        <canvas id="banChart" height="600"></canvas>
      </div>
    </div>
    <div class="flex flex-col justify-center w-full gap-4 mb-6 lg:flex-row">
      <div class="flex items-center justify-center">
        <div
          class="stats stats-vertical lg:stats-horizontal shadow mt-6 w-full mx-4 lg:mx-0"
        >
          <div class="stat text-center lg:text-start">
            <div class="stat-title">Total profiles tracked</div>
            <div class="stat-value">{{ totalProfiles }}</div>
          </div>
          <div class="stat text-center lg:text-start">
            <div class="stat-title">Total banned users</div>
            <div class="stat-value">{{ totalBanned }}</div>
          </div>
          <div class="stat text-center lg:text-start">
            <div class="stat-title">
              Banned in {{ selectedYear ?? "total" }}
            </div>
            <div class="stat-value">{{ totalBannedInYear }}</div>
          </div>
        </div>
      </div>
      <div class="flex items-center justify-center">
        <div
          class="stats stats-vertical lg:stats-horizontal shadow mt-6 w-full mx-4 lg:mx-0"
        >
          <div class="stat text-center lg:text-start">
            <div class="stat-title">VAC Ban</div>
            <div class="stat-value">{{ banTypeCounts.vacBan }}</div>
          </div>
          <div class="stat text-center lg:text-start">
            <div class="stat-title">Game Ban</div>
            <div class="stat-value">{{ banTypeCounts.gameBan }}</div>
          </div>
          <div class="stat text-center lg:text-start">
            <div class="stat-title">Trade Ban</div>
            <div class="stat-value">{{ banTypeCounts.tradeBan }}</div>
          </div>
          <div class="stat text-center lg:text-start">
            <div class="stat-title">Unban</div>
            <div class="stat-value">{{ banTypeCounts.unban }}</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
