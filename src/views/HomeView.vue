<script setup lang="ts">
import DiscourseTopicList from '@/components/DiscourseTopicList.vue'

import type {Ref} from 'vue';
import {ref, onMounted, computed} from "vue";

const config: Ref<{ "apiUrl": string, "apiRedirect": string, "categoryFilter": string }|undefined> = ref(undefined);

onMounted(async () => {
  const response = await fetch("/config.json");
  if (! response.ok)
    throw Error("ERROR: Failed to fetch configuration!")
  config.value = {...config.value, ...(await response.json())};
})

</script>

<template>
  <main>
    <DiscourseTopicList v-if="config"
                        :api-url="config.apiUrl"
                        :redirect-url="config.apiRedirect"
                        :category="config.categoryFilter"/>
  </main>
</template>
