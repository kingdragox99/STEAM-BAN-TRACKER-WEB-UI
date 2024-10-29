<script setup>
import { SpeedInsights } from "@vercel/speed-insights/vue";
</script>

<template>
  <NuxtLoadingIndicator />
  <Header />
  <NuxtPage />
  <Footer />
  <SpeedInsights />
</template>
