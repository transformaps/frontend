<template>
<b-navbar toggleable="lg" type="dark" variant="dark">
  <router-link :to="{name: 'home', params: {lang: $route.params.lang}}">
    <b-navbar-brand class="d-flex align-items-center">
      <img src="@/assets/logo.svg" alt="Logo" class="mr-2">
      <slot name="name">Aktionskarten</slot>
    </b-navbar-brand>
  </router-link>

  <b-navbar-toggle target="nav_collapse"></b-navbar-toggle>

  <b-collapse is-nav id="nav_collapse">
    <slot>
      <b-navbar-nav>
        <b-nav-item :to="{name: 'home', params: {lang: lang}}">{{$t('navbar.home')}}</b-nav-item>
        <b-nav-item :to="{name: 'tutorial', params: {lang: lang}}">{{$t('navbar.tutorial')}}</b-nav-item>
        <b-nav-item :to="{name: 'contact', params: {lang: lang}}">{{$t('navbar.contact')}}</b-nav-item>
      </b-navbar-nav>
    </slot>

    <b-navbar-nav class="ml-auto">
      <b-nav-item-dropdown :text="lang ? lang : 'en'" right>
        <b-dropdown-item :to="{name: $route.name, params: Object.assign({}, $route.params, {lang: 'en'})}">EN</b-dropdown-item>
        <b-dropdown-item :to="{name: $route.name, params: Object.assign({}, $route.params, {lang: 'de'})}">DE</b-dropdown-item>
      </b-nav-item-dropdown>

      <slot name="navbar">
      </slot>
    </b-navbar-nav>
  </b-collapse>
</b-navbar>
</template>

<script>
export default {
  name: 'navbar',
  props: ['lang']
}
</script>
