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
let allData = [];
let chartInstance = null;
const availableYears = ref([]);
const selectedYear = ref(null);
const isLoading = ref(true);
const showDetails = ref(true);
const zoomEnabled = ref(false);

const enableZoom = async () => {
  if (process.client && !zoomEnabled.value) {
    try {
      const zoomPlugin = await import("chartjs-plugin-zoom");
      Chart.register(zoomPlugin.default);
      zoomEnabled.value = true;

      if (chartInstance) {
        chartInstance.destroy();
        createChart();
      }
    } catch (error) {
      console.error("Erreur lors du chargement du plugin de zoom:", error);
    }
  }
};

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
      if (process.client) {
        enableZoom();
      }
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
        } else if (banType === "trade ban") {
          dateMap[dateOnly].trade++;
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

  const chartConfig = {
    type: "bar",
    data: {
      labels: banDates.value,
      datasets: [
        {
          label: "VAC Ban",
          data: banCounts.value.vacBan,
          backgroundColor: "rgba(233, 46, 144, 0.8)",
          borderColor: "rgba(233, 46, 144, 1)",
          borderWidth: 1,
          borderRadius: 4,
          barPercentage: 0.8,
          categoryPercentage: 0.8,
        },
        {
          label: "Game Ban",
          data: banCounts.value.gameBan,
          backgroundColor: "rgba(21, 91, 224, 0.8)",
          borderColor: "rgba(21, 91, 224, 1)",
          borderWidth: 1,
          borderRadius: 4,
          barPercentage: 0.8,
          categoryPercentage: 0.8,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      layout: {
        padding: {
          top: 20,
          right: 25,
          bottom: 20,
          left: 25,
        },
      },
      scales: {
        y: {
          beginAtZero: true,
          grid: {
            color: getGridColor(),
            drawBorder: true,
            lineWidth: 0.5,
          },
          ticks: {
            color: getTextColor(),
            precision: 0,
            font: {
              size: 14,
              weight: "bold",
            },
            padding: 10,
            title: {
              display: true,
              text: "Number of bans",
              color: getTextColor(),
              font: {
                size: 16,
                weight: "bold",
              },
              padding: {
                bottom: 10,
              },
            },
          },
        },
        x: {
          grid: {
            color: getGridColor(),
            drawBorder: true,
            lineWidth: 0.5,
            display: false,
          },
          ticks: {
            color: getTextColor(),
            maxRotation: 45,
            minRotation: 45,
            autoSkip: true,
            maxTicksLimit: 15,
            font: {
              size: 14,
              weight: "bold",
            },
            padding: 10,
            callback: function (value, index, values) {
              const date = new Date(this.getLabelForValue(value));
              return formatDate(date);
            },
            title: {
              display: true,
              text: "Date",
              color: getTextColor(),
              font: {
                size: 16,
                weight: "bold",
              },
              padding: {
                top: 20,
              },
            },
          },
        },
      },
      plugins: {
        legend: {
          display: true,
          position: "top",
          align: "center",
          labels: {
            color: getTextColor(),
            font: {
              size: 16,
              weight: "bold",
            },
            padding: 20,
            usePointStyle: true,
            pointStyle: "circle",
            boxWidth: 10,
          },
        },
        tooltip: {
          backgroundColor: "rgba(0, 0, 0, 0.8)",
          titleFont: {
            size: 16,
            weight: "bold",
          },
          bodyFont: {
            size: 14,
          },
          padding: 12,
          cornerRadius: 6,
          displayColors: true,
          callbacks: {
            title: (tooltipItems) => {
              const date = new Date(tooltipItems[0].label);
              return formatDate(date, true);
            },
            label: (context) => {
              let label = context.dataset.label || "";
              if (label) {
                label += ": ";
              }
              if (context.parsed.y !== null) {
                label += context.parsed.y;
              }
              return label;
            },
          },
        },
      },
    },
  };

  if (zoomEnabled.value) {
    chartConfig.options.plugins.zoom = {
      pan: {
        enabled: true,
        mode: "xy",
      },
      zoom: {
        wheel: {
          enabled: true,
        },
        pinch: {
          enabled: true,
        },
        mode: "xy",
      },
      limits: {
        y: { min: 0 },
      },
    };
  }

  chartInstance = new Chart(ctx, chartConfig);
};

const updateChart = () => {
  if (chartInstance) {
    chartInstance.data.labels = banDates.value;
    chartInstance.data.datasets[0].data = banCounts.value.vacBan;
    chartInstance.data.datasets[1].data = banCounts.value.gameBan;
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

const resetZoom = () => {
  if (chartInstance && zoomEnabled.value) {
    chartInstance.resetZoom();
  }
};

const formatDate = (date, longFormat = false) => {
  if (longFormat) {
    return date.toLocaleDateString(undefined, {
      year: "numeric",
      month: "long",
      day: "numeric",
    });
  } else {
    if (banDates.value.length > 90) {
      return date.toLocaleDateString(undefined, {
        month: "short",
        year: "numeric",
      });
    } else if (banDates.value.length > 30) {
      return date.toLocaleDateString(undefined, {
        day: "numeric",
        month: "short",
      });
    } else {
      return date.toLocaleDateString(undefined, {
        day: "numeric",
        month: "short",
        year: "2-digit",
      });
    }
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
        <h2 class="text-2xl font-bold text-center mb-4">SBT Charts</h2>
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
      <div
        class="w-full flex flex-col sm:flex-row justify-between items-center px-4 gap-2 mb-2"
      >
        <button
          @click="showDetails = !showDetails"
          :class="[
            'btn',
            showDetails === true ? 'btn-active' : 'btn-outline',
            'btn-secondary',
          ]"
        >
          {{ showDetails ? "Hide Details" : "Show Details" }}
        </button>
        <button
          v-if="zoomEnabled"
          @click="resetZoom()"
          class="btn btn-outline btn-primary"
        >
          Reset Zoom
        </button>
      </div>
      <div
        class="w-full px-4 sm:px-6 lg:px-8 relative bg-base-200 rounded-lg p-4 shadow-lg"
      >
        <canvas id="banChart" height="600"></canvas>
        <div
          v-show="showDetails"
          class="absolute top-14 right-3 text-lg bg-base-100 p-4 rounded-md shadow-lg flex flex-col gap-2 sm:top-10 sm:right-10"
        >
          <p class="font-bold text-secondary text-xl">
            Total banned: {{ totalBansForSelectedYear }}
          </p>
          <p class="font-bold text-lg" style="color: rgba(233, 46, 144, 1)">
            VAC Ban: {{ vacBanCount }}
          </p>
          <p class="font-bold text-lg" style="color: rgba(21, 91, 224, 1)">
            Game Ban: {{ gameBanCount }}
          </p>
        </div>
      </div>
    </div>
  </div>
</template>
