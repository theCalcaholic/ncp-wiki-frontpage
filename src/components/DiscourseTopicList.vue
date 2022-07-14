<script setup lang="ts">
import type {Ref} from "vue";
import {ref, computed, onMounted} from "vue";
import DiscourseTopic from "./DiscourseTopic.vue"
import logoUrl from "../assets/logo.svg";

const props = defineProps<{
  apiUrl: string,
  redirectUrl: string
  category: string
}>()

const topics: Ref<Map<string, Array<any>>> = ref(new Map<string, Array<Object>>());
const topTags: Ref<Array<string>> = ref([]);
const sections: Ref<Array<string>> = ref([])
const selectedTags: Ref<Array<string>> = ref([]);
const searchPhrase: Ref<string> = ref("");
const searchResults: Ref<Map<string, Array<any>>> = ref(new Map<string, Array<Object>>());
const searchState: Ref<'IDLE' | 'RUNNING' | 'COMPLETE'> = ref('IDLE');
const showFilters: Ref<boolean> = ref(true);

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

async function performSearch(query: string) {
  if (query.length === 0) {
    searchState.value = 'IDLE';
    searchResults.value.clear();
    return
  }

  let uri = `${props.apiUrl}/search.json?q=` + encodeURIComponent(`${query} #${props.category}`);
  console.log(`Fetching ${uri}...`);
  searchState.value = 'RUNNING';
  let response = await fetch(uri);
  if (!response.ok)
    throw new Error("Failed to fetch search results!")
  let data = await response.json();
  searchResults.value.set("Search Results", data.topics ?? []);
  searchState.value = 'COMPLETE'
}

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

  <div :class="`filter-container${showFilters ? '' : ' hidden'}`">
    <div :class="`filter-controller`" @click="showFilters = !showFilters"></div>
    <div :class="`search`">
      <h4>Search</h4>
      <input type="text" v-model="searchPhrase" @input="performSearch(searchPhrase)">
    </div>
    <div :class="`tags`">
      <h4>Filter by tag:</h4>
      <ul class="taglist">
        <template v-for="tag in topTags">
          <li v-bind:class="`discourse-tag${selectedTags.includes(tag) ? ' selected' : ' '}`"
              v-on:click="toggleSelectTag(tag)">
            {{ tag }}
          </li>
        </template>
      </ul>
    </div>
  </div>
  <h2>Articles</h2>
  <ul>
    <li class="section" v-if="searchState === 'RUNNING' || filteredTopics.size === 0">
      Loading...
    </li>
    <template v-else v-for="section in (searchState === 'COMPLETE' ? searchResults : filteredTopics)">
      <li class="section" v-if="section[1].length !== 0 || searchState === 'COMPLETE'">
        <h2 >{{ section[0] }}</h2>
        <ul v-for="topic in section[1]">
          <DiscourseTopic v-if="matchesFilter(topic.tags)" :base-url="redirectUrl" :topic="topic"></DiscourseTopic>
        </ul>
      </li>
    </template>
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
  padding-left: 0;
}

.section {
  margin-bottom: 2em;
  margin-top: 1em;
  background-color: #2c3e5080;
  padding: 1em;
}

.filter-container {
  display: flex;
  flex-direction: column;
  gap: 1em;
  align-content: space-evenly;
  padding-bottom: 1em;
  margin-bottom: 1em;
  border-bottom: white double thin;
  transition: gap .25s;
}

.filter-controller {
  width: 32px;
  height: 32px;
  background-image: url(../assets/logo.svg);
  flex-shrink: 0;
  flex-grow: 0;
  cursor: pointer;
  align-self: end;
}

.filter-container.hidden .filter-controller {
  transform: rotate(180deg);
}


.search, .tags {
  width: 100%;
  flex-grow: 1;
  flex-shrink: 1;
  gap: 1em;
  overflow: hidden;
  opacity: 1;
  transition-timing-function: linear;
  transition: font-size .25s,
  margin .25s,
  padding .25s,
  opacity .5s .25s;
}

.filter-container.hidden .search, .filter-container.hidden .tags {
  font-size: 0;
  margin: 0;
  padding: 0;
  opacity: 0;
  transition: opacity .25s,
  font-size .5s .25s,
  margin .5s .25s,
  padding .5s .25s,
  gap .5s .25s;
}

.filter-container.hidden {
  gap: 0;
  transition: gap .5s .25s;
}

.search {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.search input {
  width: 100%;
  height: 2em;
  line-height: 2em;
  background-color: #6588aa;
  border: white 0 solid;
  border-radius: 10px;
  color: white;
  font-weight: bold;
  font-size: 1.2em;
  padding: 3px 6px;
}

@media (min-width: 1024px) {
  .filter-container {
    flex-direction: row;
  }

  .filter-controller {
    align-self: start;
  }
}


</style>