<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router';
import type {Ref} from "vue";
import {computed, ref} from "vue";

const logoLightUrl = new URL('./assets/ncp-logo-dark.svg', import.meta.url);
const logoDarkUrl = new URL('./assets/ncp-logo-light.svg', import.meta.url);

const darkThemeQuery = window.matchMedia("(prefers-color-scheme: dark)");

const logoUrl: Ref<URL> = ref(darkThemeQuery.matches ? logoDarkUrl : logoLightUrl);

darkThemeQuery.addEventListener("change", e => {
  if (e.matches)
    logoUrl.value = logoDarkUrl
  else
    logoUrl.value = logoLightUrl
})

</script>

<template>
<!--  <header>-->

<!--    <div class="wrapper">-->

<!--      <nav>-->
<!--        <RouterLink to="/">Home</RouterLink>-->
<!--        <RouterLink to="/about">About</RouterLink>-->
<!--      </nav>-->
<!--    </div>-->
<!--  </header>-->

  <header><h1>NextcloudPi Documentation</h1><img class="logo" :src="logoUrl.toString()" alt="Logo"></header>
  <RouterView />
</template>

<style scoped>
header {
  line-height: 1.5;
  max-height: 100vh;
  margin-bottom: 4em;
  display: flex;
}

.logo {
  display: block;
  width: 4rem;
  flex-shrink: 0;
  height: 4rem;
}
.header h1 {
  flex-shrink: 1;
}

nav {
  width: 100%;
  font-size: 12px;
  text-align: center;
  margin-top: 2rem;
}

nav a.router-link-exact-active {
  color: var(--color-text);
}

nav a.router-link-exact-active:hover {
  background-color: transparent;
}

nav a {
  display: inline-block;
  padding: 0 1rem;
  border-left: 1px solid var(--color-border);
}

nav a:first-of-type {
  border: 0;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    justify-content: space-between;

  }

  .logo {
    margin: 0 2rem 0 2rem;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }

  nav {
    text-align: left;
    margin-left: -1rem;
    font-size: 1rem;

    padding: 1rem 0;
    margin-top: 1rem;
  }
}
</style>
