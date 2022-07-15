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
    <div class="filter-controller-container">
      <div class="filter-controller" @click="showFilters = !showFilters"></div>
      <div class="filter-controller-label">Search & Filter</div>
    </div>
    <div class="search">
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
  <ul class="sections">
    <li class="section" v-if="searchState === 'RUNNING' || filteredTopics.size === 0">
      Loading...
    </li>
    <template v-else v-for="section in (searchState === 'COMPLETE' ? searchResults : filteredTopics)">
      <li class="section" v-if="section[1].length !== 0 || searchState === 'COMPLETE'">
        <h2 >{{ section[0] }}</h2>
        <ul class="topics" v-for="topic in section[1]">
          <DiscourseTopic v-if="matchesFilter(topic.tags)" :base-url="redirectUrl" :topic="topic"></DiscourseTopic>
        </ul>
      </li>
    </template>
  </ul>
</template>

<style>

li.discourse-tag {
  display: inline-block;
  background-color: var(--color-foreground);
  cursor: pointer;
  list-style: none;
  padding: 4px;
  margin: 2px;
  color: var(--color-text-contrast);
}
li.discourse-tag.selected {
  background-color: var(--color-foreground-accent)
}

ul.taglist {
  display: flex;
  flex-direction: row;
  flex-flow: wrap;
  padding-left: 0;
}
ul.sections, ul.topics {
  list-style: none;
  padding-left: 0;
}

.section {
  margin-bottom: 2em;
  margin-top: 1em;
  background-color: rgba(var(--color-foreground-raw), 0.5);
  padding: 1em;
}

.filter-container {
  display: flex;
  flex-direction: column;
  gap: 1em;
  align-content: space-evenly;
  padding-bottom: 1em;
  margin-bottom: 1em;
  border-bottom: var(--color-border-hover) double thin;
  transition: gap .25s;
}

.filter-controller {
  width: calc(40px - 1em);
  height: calc(40px - 1em);
  background-image: var(--dropdown-logo);
  background-size: contain;
  flex-shrink: 0;
  flex-grow: 0;
  cursor: pointer;
  transition: transform .25s;
}

.filter-container.hidden .filter-controller {
  transform: rotate(90deg);
  transition: transform .5s;
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
  background-color: var(--color-foreground-light);
  border: var(--color-foreground) 3px solid;
  border-radius: 10px;
  color: var(--color-text);
  font-weight: bold;
  font-size: 1.2em;
  padding: 3px 6px;
  outline: none;
}



.filter-container .filter-controller-container {
  display: flex;
  flex-direction: row-reverse;

}

.filter-container .filter-controller-label {
  visibility: hidden;
  width: 0;
  margin-right: 0;
  opacity: 0;
  transition: opacity .25s;
  white-space: nowrap;
  text-align: right;
}

.filter-container.hidden .filter-controller-label {
  visibility: visible;
  width: calc(100% - var(--dropdown-button-size));
  float: right;
  margin-right: 1em;
  color: var(--color-text);
  opacity: 1;
  transition: opacity .5s .25s, visibility .5s 0s;
}

@media (min-width: 1024px) {
  ul.sections, ul.topics {
    padding-left: 40px;
  }
  .filter-container {
    flex-direction: row-reverse;
  }
  .filter-container.hidden .filter-controller-container {
    width: 100%;
    transition: width 1s .25s;
  }
  .filter-container .filter-controller-container {
    width: var(--dropdown-button-size);
    height: var(--dropdown-button-size);
    transition: width .25s;
    float: right;
    flex-shrink: 2;
  }
  .filter-container.hidden .filter-controller {
    float: right;
  }
}


</style>