<script setup>
import { createClient } from "@supabase/supabase-js";
import { onMounted, ref } from "vue";

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

const totalProfiles = ref(0);
const totalBanned = ref(0);
const banTypeCounts = ref({ vacBan: 0, gameBan: 0, tradeBan: 0, unban: 0 });
const isLoading = ref(true);

const fetchProfileData = async () => {
  try {
    const { count: totalProfilesCount, error: profilesError } = await supabase
      .from("profil")
      .select("id", { count: "exact" });

    if (profilesError) {
      console.error(
        "Erreur lors de la récupération du nombre total de profils: ",
        profilesError
      );
      throw new Error(
        "Erreur lors de la récupération du nombre total de profils: " +
          profilesError.message
      );
    }

    totalProfiles.value = totalProfilesCount;

    const { count: totalBannedCount, error: bannedError } = await supabase
      .from("profil")
      .select("id", { count: "exact" })
      .eq("ban", true);

    if (bannedError) {
      console.error(
        "Erreur lors de la récupération du nombre de profils bannis: ",
        bannedError
      );
      throw new Error(
        "Erreur lors de la récupération du nombre de profils bannis: " +
          bannedError.message
      );
    }

    totalBanned.value = totalBannedCount;

    const counts = { vacBan: 0, gameBan: 0, tradeBan: 0, unban: 0 };
    let start = 0;
    const limit = 1000;
    let fetchMore = true;

    while (fetchMore) {
      const { data, error } = await supabase
        .from("profil")
        .select("ban, ban_type")
        .eq("ban", true)
        .range(start, start + limit - 1);

      if (error) {
        console.error(
          "Erreur lors de la récupération des types de ban: ",
          error
        );
        throw new Error(
          "Erreur lors de la récupération des types de ban: " + error.message
        );
      }

      if (data && data.length > 0) {
        data.forEach((profile) => {
          if (profile.ban_type) {
            const type = profile.ban_type.trim().toLowerCase();
            if (type === "vac ban") {
              counts.vacBan++;
            } else if (type === "game ban") {
              counts.gameBan++;
            } else if (type === "trade ban") {
              counts.tradeBan++;
            } else if (type === "unban") {
              counts.unban++;
            }
          }
        });
      }

      if (!data || data.length < limit) {
        fetchMore = false;
      } else {
        start += limit;
      }
    }

    banTypeCounts.value = counts;
  } catch (err) {
    console.error("Erreur lors de la récupération des données: ", err);
  } finally {
    isLoading.value = false;
  }
};

onMounted(() => {
  fetchProfileData();
});
</script>

<template>
  <div class="flex flex-col justify-center w-full gap-4 mb-6 lg:flex-row">
    <div class="flex items-center justify-center">
      <div
        v-if="isLoading"
        class="stats stats-vertical lg:stats-horizontal shadow mt-6 w-full mx-4 lg:mx-0 animate-pulse"
      >
        <div class="stat text-center lg:text-start">
          <div class="stat-title animate-pulse">Loading...</div>
          <div class="stat-value animate-pulse">---</div>
        </div>
        <div class="stat text-center lg:text-start">
          <div class="stat-title animate-pulse">Loading...</div>
          <div class="stat-value animate-pulse">---</div>
        </div>
      </div>
      <div
        v-else
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
      </div>
    </div>
    <div class="flex items-center justify-center">
      <div
        v-if="isLoading"
        class="stats stats-vertical lg:stats-horizontal shadow mt-6 w-full mx-4 lg:mx-0 animate-pulse"
      >
        <div class="stat text-center lg:text-start">
          <div class="stat-title animate-pulse">Loading...</div>
          <div class="stat-value animate-pulse">---</div>
        </div>
        <div class="stat text-center lg:text-start">
          <div class="stat-title animate-pulse">Loading...</div>
          <div class="stat-value animate-pulse">---</div>
        </div>
      </div>
      <div
        v-else
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
      </div>
    </div>
  </div>
</template>
