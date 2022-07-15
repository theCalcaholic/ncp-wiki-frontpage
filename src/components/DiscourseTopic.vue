<script setup lang="ts">
import type {ComputedRef} from "vue";
import {computed} from "vue";

const props = defineProps({
  baseUrl: {type: String, required: true},
  topic: {required: true, type: Object}
})
const tags: ComputedRef<Array<string>> = computed(() => {
  if (props.topic.tags)
    return props.topic.tags
  return []
})
</script>

<template>
  <li class="topic">
    <a v-bind:href="`${baseUrl}/t/${props.topic.slug}`">
    <h3>{{ props.topic.title }}</h3>
<!--    <img v-if="props.topic.image_url" v-bind:src="props.topic.image_url"/>-->
    <p>{{ props.topic.excerpt }}</p>
    </a>
  </li>
</template>

<style>
li.topic {
  list-style: none;
  margin-top: 2em;
  border-top: white dotted thin;
}

.topic h3 {
  margin-bottom: .5em;
}
</style>