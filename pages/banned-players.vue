<script setup>
import { createClient } from "@supabase/supabase-js";
import { ref, watch, computed } from "vue";

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

// States
const bannedPlayers = ref([]);
const isLoading = ref(true);
const searchQuery = ref(""); // For ID numeric search
const steamSearchInput = ref(""); // For Steam URL/ID64 search
const searchError = ref("");
const currentPage = ref(1);
const itemsPerPage = 10;
const totalItems = ref(0);
const totalPages = computed(() => Math.ceil(totalItems.value / itemsPerPage));

// Fetch banned players with pagination
const fetchBannedPlayers = async (page = 1) => {
  isLoading.value = true;
  currentPage.value = page;

  try {
    let query = supabase
      .from("profil")
      .select("id", { count: "exact" })
      .eq("ban", true)
      .not("ban_type", "ilike", "%trade ban%");

    // Add ID filter if search query is provided and is a valid number
    if (searchQuery.value && !isNaN(parseInt(searchQuery.value))) {
      // If the search is a valid number, filter by ID
      const searchId = parseInt(searchQuery.value);
      query = query.eq("id", searchId);
    }

    const { count, error: countError } = await query;

    if (countError) {
      console.error("Error getting count:", countError);
      return;
    }

    totalItems.value = count || 0;

    // Then get data for current page
    const start = (page - 1) * itemsPerPage;
    const end = start + itemsPerPage - 1;

    let dataQuery = supabase
      .from("profil")
      .select("id, steam_name, url, ban_type, ban_date, status")
      .eq("ban", true)
      .not("ban_type", "ilike", "%trade ban%");

    // Add the same ID filter to data query
    if (searchQuery.value && !isNaN(parseInt(searchQuery.value))) {
      const searchId = parseInt(searchQuery.value);
      dataQuery = dataQuery.eq("id", searchId);
    }

    // Add pagination and ordering
    dataQuery = dataQuery
      .range(start, end)
      .order("ban_date", { ascending: false });

    const { data, error } = await dataQuery;

    if (error) {
      console.error("Error fetching banned players:", error);
      return;
    }

    bannedPlayers.value = data || [];
  } catch (error) {
    console.error("Error in fetchBannedPlayers:", error);
  } finally {
    isLoading.value = false;
  }
};

// Handle search by Steam ID or URL
const handleSteamSearch = async () => {
  searchError.value = "";
  if (!steamSearchInput.value.trim()) {
    // Reset to normal view if search is empty
    fetchBannedPlayers(1);
    return;
  }

  isLoading.value = true;
  try {
    // Extract Steam ID from URL or use direct ID
    const steamId = extractSteamID64(steamSearchInput.value);

    if (!steamId) {
      searchError.value = "Invalid Steam URL or ID format";
      isLoading.value = false;
      return;
    }

    // Search for URL column containing this Steam ID
    let { data, error } = await supabase
      .from("profil")
      .select("id, steam_name, url, ban_type, ban_date, status, ban")
      .ilike("url", `%${steamId}%`)
      .eq("ban", true)
      .not("ban_type", "ilike", "%trade ban%");

    if (error) {
      console.error("Error searching by Steam ID:", error);
      searchError.value = "Error processing search";
      isLoading.value = false;
      return;
    }

    // If nothing found, try checking in the status field which might contain the Steam ID
    if (!data || data.length === 0) {
      // We'll try to search in the status field, if it contains JSON data
      const { data: statusMatchData, error: statusError } = await supabase
        .from("profil")
        .select("id, steam_name, url, ban_type, ban_date, status, ban")
        .not("status", "is", null)
        .eq("ban", true)
        .not("ban_type", "ilike", "%trade ban%");

      if (!statusError && statusMatchData && statusMatchData.length > 0) {
        // Filter the results to find entries where status JSON contains the Steam ID
        const matchedPlayers = statusMatchData.filter((player) => {
          try {
            const statusObj = JSON.parse(player.status);
            // Check various fields that might contain the Steam ID
            const fieldsToCheck = ["steamid", "steamID64", "profileurl"];
            return fieldsToCheck.some(
              (field) => statusObj[field] && statusObj[field].includes(steamId)
            );
          } catch (e) {
            return false;
          }
        });

        if (matchedPlayers.length > 0) {
          data = matchedPlayers;
        }
      }
    }

    if (!data || data.length === 0) {
      searchError.value = "No banned players found with this Steam ID";
      isLoading.value = false;
      return;
    }

    // Show the results
    bannedPlayers.value = data;
    totalItems.value = data.length;
    currentPage.value = 1;
  } catch (error) {
    console.error("Error in Steam search:", error);
    searchError.value = "Error processing search";
  } finally {
    isLoading.value = false;
  }
};

// Function to extract SteamID64 from various formats
const extractSteamID64 = (input) => {
  // Remove any whitespace
  const trimmed = input.trim();

  // Check if it's already a SteamID64 (17-digit number)
  if (/^\d{17}$/.test(trimmed)) {
    return trimmed;
  }

  // Check for profile URLs with steamID64 in them
  const profileMatch = trimmed.match(
    /steamcommunity\.com\/profiles\/(\d{17})/i
  );
  if (profileMatch && profileMatch[1]) {
    return profileMatch[1];
  }

  // Check for custom URL profiles
  const customUrlMatch = trimmed.match(/steamcommunity\.com\/id\/([^\/]+)/i);
  if (customUrlMatch && customUrlMatch[1]) {
    // We'll just return the custom URL part, the search will look for it in the URL
    return customUrlMatch[1];
  }

  return null;
};

// Format a date nicely
const formatDate = (dateString) => {
  if (!dateString) return "Unknown";

  const date = new Date(dateString);
  return date.toLocaleDateString(undefined, {
    year: "numeric",
    month: "short",
    day: "numeric",
  });
};

// Generate Steam profile URL from the url field or using the ID
const getSteamProfileUrl = (player) => {
  if (player.url) return player.url;
  return `https://steamcommunity.com/profiles/${player.id}`;
};

// Format ban id for display
const formatBanId = (id) => {
  return `Player #${id}`;
};

// Navigation functions
const goToPage = (page) => {
  if (page >= 1 && page <= totalPages.value) {
    fetchBannedPlayers(page);
  }
};

const goToPrevPage = () => {
  if (currentPage.value > 1) {
    fetchBannedPlayers(currentPage.value - 1);
  }
};

const goToNextPage = () => {
  if (currentPage.value < totalPages.value) {
    fetchBannedPlayers(currentPage.value + 1);
  }
};

const goBack10Pages = () => {
  const targetPage = Math.max(1, currentPage.value - 10);
  fetchBannedPlayers(targetPage);
};

const goForward10Pages = () => {
  const targetPage = Math.min(totalPages.value, currentPage.value + 10);
  fetchBannedPlayers(targetPage);
};

// Refresh results when search query changes
watch(searchQuery, () => {
  fetchBannedPlayers(1);
});

// Reset search when Steam search input is cleared
watch(steamSearchInput, (newValue) => {
  if (!newValue.trim()) {
    searchError.value = "";
    // If we used Steam search before and now cleared it, reload all players
    if (
      bannedPlayers.value.length < totalItems.value ||
      totalItems.value === 0
    ) {
      fetchBannedPlayers(1);
    }
  }
});

// On component mount, fetch initial data
fetchBannedPlayers();

// Get the avatar URL for a player
const getAvatarUrl = (player) => {
  // Try to extract avatar from the status field which might contain JSON data with avatar
  if (player.status) {
    try {
      const statusData = JSON.parse(player.status);
      if (statusData.avatarfull) return statusData.avatarfull;
      if (statusData.avatarmedium) return statusData.avatarmedium;
      if (statusData.avatar) return statusData.avatar;
    } catch (e) {
      // If parsing fails, status is not a valid JSON
    }
  }

  // Default avatar if no image URL is found
  return `https://avatars.steamstatic.com/fef49e7fa7e1997310d705b2a6158ff8dc1cdfeb_full.jpg`;
};
</script>

<template>
  <div class="container mx-auto px-4 py-8">
    <h1 class="text-3xl font-bold mb-8 text-center">Banned Players List</h1>

    <!-- Search section -->
    <div class="mb-8">
      <div class="flex flex-col md:flex-row gap-4 mb-6">
        <!-- Regular ID search -->
        <div class="flex-1">
          <div class="form-control">
            <label class="label">
              <span class="label-text font-medium">Search by Player ID</span>
            </label>
            <div class="flex gap-2">
              <input
                v-model="searchQuery"
                type="number"
                placeholder="Enter exact player ID..."
                class="input input-bordered w-full"
                @keyup.enter="fetchBannedPlayers(1)"
              />
              <button
                v-if="searchQuery"
                class="btn btn-ghost"
                @click="searchQuery = ''"
                title="Clear search"
              >
                ×
              </button>
            </div>
          </div>
        </div>

        <!-- Steam ID/URL search -->
        <div class="flex-1">
          <div class="form-control">
            <label class="label">
              <span class="label-text font-medium">Search by Steam ID/URL</span>
            </label>
            <div class="flex gap-2">
              <input
                v-model="steamSearchInput"
                type="text"
                placeholder="Enter Steam ID or profile URL..."
                class="input input-bordered w-full"
                @keyup.enter="handleSteamSearch"
              />
              <button
                v-if="steamSearchInput"
                class="btn btn-ghost"
                @click="steamSearchInput = ''"
                title="Clear search"
              >
                ×
              </button>
              <button
                class="btn btn-primary"
                @click="handleSteamSearch"
                :disabled="isLoading"
              >
                <span
                  v-if="isLoading"
                  class="loading loading-spinner loading-xs"
                ></span>
                Search
              </button>
            </div>
            <label v-if="searchError" class="label">
              <span class="label-text-alt text-error">{{ searchError }}</span>
            </label>
          </div>
        </div>
      </div>
    </div>

    <!-- Results section -->
    <div class="bg-base-200 p-4 rounded-lg shadow-lg">
      <div v-if="isLoading" class="flex justify-center py-12">
        <div class="loading loading-spinner loading-lg text-primary"></div>
      </div>

      <div v-else-if="bannedPlayers.length === 0" class="text-center py-12">
        <p class="text-xl">No banned players found.</p>
      </div>

      <div v-else>
        <!-- Players table -->
        <div class="overflow-x-auto">
          <table class="table w-full">
            <thead>
              <tr>
                <th>Player</th>
                <th>Steam Profile</th>
                <th>Ban Type</th>
                <th>Ban Date</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="player in bannedPlayers"
                :key="player.id"
                class="hover"
              >
                <td>
                  <div class="flex items-center space-x-3">
                    <div class="avatar">
                      <div class="mask mask-squircle w-12 h-12">
                        <img
                          :src="getAvatarUrl(player)"
                          :alt="player.steam_name || 'Player'"
                        />
                      </div>
                    </div>
                    <div>
                      <div class="font-bold">
                        {{ player.steam_name || "Unknown Player" }}
                      </div>
                      <div class="text-sm opacity-50">ID: {{ player.id }}</div>
                    </div>
                  </div>
                </td>
                <td>
                  <a
                    :href="getSteamProfileUrl(player)"
                    target="_blank"
                    class="link link-primary"
                  >
                    View Profile
                  </a>
                </td>
                <td>
                  <div
                    class="badge"
                    :class="{
                      'badge-error': player.ban_type
                        ?.toLowerCase()
                        .includes('vac'),
                      'badge-warning': player.ban_type
                        ?.toLowerCase()
                        .includes('game'),
                    }"
                  >
                    {{ player.ban_type || "Unknown" }}
                  </div>
                </td>
                <td>{{ formatDate(player.ban_date) }}</td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Pagination with arrows -->
        <div
          class="flex justify-center items-center gap-4 mt-6"
          v-if="totalPages > 1"
        >
          <div class="join shadow-md">
            <button
              class="join-item btn"
              :disabled="currentPage === 1"
              @click="goBack10Pages()"
              title="Go back 10 pages"
            >
              &laquo;
            </button>
            <button
              class="join-item btn"
              :disabled="currentPage === 1"
              @click="goToPrevPage()"
              title="Previous page"
            >
              &lsaquo;
            </button>
            <button class="join-item btn">
              Page {{ currentPage }} of {{ totalPages }}
            </button>
            <button
              class="join-item btn"
              :disabled="currentPage === totalPages"
              @click="goToNextPage()"
              title="Next page"
            >
              &rsaquo;
            </button>
            <button
              class="join-item btn"
              :disabled="currentPage === totalPages"
              @click="goForward10Pages()"
              title="Go forward 10 pages"
            >
              &raquo;
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
