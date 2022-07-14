<script setup lang="ts">
import type {Ref} from "vue";
import {ref, computed, onMounted} from "vue";
import DiscourseTopic from "./DiscourseTopic.vue"

const props = defineProps<{
  apiUrl: string,
  redirectUrl: string
  category: string
}>()

const topics: Ref<Map<string, Array<any>>> = ref(new Map<string, Array<Object>>());
const topTags: Ref<Array<string>> = ref([]);
const sections: Ref<Array<string>> = ref([])
const selectedTags: Ref<Array<string>> = ref([]);

const regexStartsWithNumber = /^\d-/

onMounted(async () => {

  let response = await fetch(`${props.apiUrl}/c/${props.category}.json`);
  if (!response.ok)
    throw new Error("Failed to fetch discourse posts!");
  let data = await response.json();
  topTags.value = data.topic_list.top_tags.filter((t: string) => !regexStartsWithNumber.test(t)).sort();
  sections.value = data.topic_list.top_tags.filter((t: string) => regexStartsWithNumber.test(t)).sort();
  topics.value = sections.value.reduce((map: Map<string, Array<Object>>, section: string) => {
    const sectionTitle = section
        .replace(regexStartsWithNumber, "")
        .split(/[_ ]/)
        .map(s => s[0].toUpperCase() + s.substring(1))
        .join(" ");
    map.set(sectionTitle, data.topic_list.topics.filter((topic: any) => topic.tags.includes(section)));
    return map;
  }, new Map<string, Array<Object>>());
  topics.value.set("Other", data.topic_list.topics
      .filter((topic: any) => topic.tags.every((t: string) => !sections.value.includes(t))));
});

function toggleSelectTag(tag: string) {
  if (selectedTags.value.includes(tag))
    selectedTags.value.splice(selectedTags.value.indexOf(tag), 1)
  else
    selectedTags.value.push(tag)
}

function matchesFilter(tags: Array<string>) {
  return selectedTags.value.length == 0 || selectedTags.value.every(tags.includes.bind(tags))
}

const filteredTopics = computed(() => {
  let filtered = new Map<string, Array<any>>();
  for (let [key, entry] of topics.value.entries()) {
    filtered.set(key, entry.filter(e => matchesFilter(e.tags)));
  }
  return filtered;
})

</script>

<template>
  <h4>Filter by tag:</h4>
  <ul class="taglist">
    <template v-for="tag in topTags">
      <li v-bind:class="`discourse-tag${selectedTags.includes(tag) ? ' selected' : ' '}`"
          v-on:click="toggleSelectTag(tag)">
        {{ tag }}
      </li>
    </template>
  </ul>
  <h2>Articles</h2>
  <ul>
    <div class="section" v-for="section in filteredTopics">
      <h2 v-if="section[1].length !== 0">{{ section[0] }}</h2>
      <template v-for="topic in section[1]">
        <DiscourseTopic v-if="matchesFilter(topic.tags)" :base-url="redirectUrl" :topic="topic"></DiscourseTopic>
      </template>
    </div>
  </ul>
</template>

<style>

li.discourse-tag {
  display: inline-block;
  background-color: #2c3e50;
  cursor: pointer;
  list-style: none;
  padding: 4px;
  margin: 2px;
}
li.discourse-tag.selected {
  background-color: #2c8050
}

ul.taglist {
  display: flex;
  flex-direction: row;
  flex-flow: wrap;
  border-bottom: white double thin;
}

.section {
  margin-bottom: 2em;
  margin-top: 1em;
  background-color: #2c3e5080;
}

</style>