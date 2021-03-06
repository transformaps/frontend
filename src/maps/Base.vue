<template>
  <div class="w-100 h-100">
    <navbar :lang="lang">
      <template slot="name" v-if="model">
        {{model.name}}
      </template>
      <template v-if="model">
        <div v-if="model.authenticated">
          <b-badge v-if="model.published" variant="success">{{$t('navbar.published')}}</b-badge>
          <b-badge v-else variant="warning">{{$t('navbar.private')}}</b-badge>
        </div>

        <b-navbar-nav class="d-lg-none">
          <b-nav-item :to="{name: 'map.edit', params: {id: model.id, secret: secret}}">
            {{$t('navsteps.form')}}
          </b-nav-item>
          <b-nav-item v-if="model.authenticated && secret" :to="{name: 'map.bbox', params: {id: model.id, secret: secret}}">
            {{$t('navsteps.bbox')}}
          </b-nav-item>
          <b-nav-item :to="{name: 'map', params: {id: model.id, secret: secret}}">
            {{$t('navsteps.map.' + (model && model.authenticated ? 'edit' : 'static') )}}
          </b-nav-item>
          <b-nav-item :to="{name: 'map.preview', params: {id: model.id, secret: secret}}">
            {{$t(model.authenticated ? 'navsteps.preview' : 'navsteps.download')}}
          </b-nav-item>
        </b-navbar-nav>
      </template>

      <template slot="navbar" v-if="model">
        <b-navbar-nav class="ml-md-auto" v-if="model">
          <b-nav-form>
            <b-btn  v-if="!model.authenticated" size="sm" variant="primary"  v-b-modal.modalLogin>
              {{$t('navbar.login')}}
            </b-btn>
            <b-btn  v-if="model.authenticated" size="sm" variant="primary"  @click="logout()">
              {{$t('navbar.logout')}}
            </b-btn>
          </b-nav-form>
        </b-navbar-nav>
      </template>
    </navbar>

    <modal-disconnected :visible="showModalDisconnected" :model="model" :lang="lang" v-if="model"></modal-disconnected>

    <b-modal id="modalLogin" ref="modalLogin" size="sm" :title="$t('navbar.authorization')" :ok-title="$t('navbar.login')" @ok="tryLogin" centered>
      <div class="container">
        <p v-if="inputSecretMsg" class="text-danger">{{inputSecretMsg}}</p>
        <b-form @submit.stop.prevent="login">
           <b-form-input id="inputToken" :state="inputSecretState" placeholder="Admin Token" v-model.trim="inputSecret"></b-form-input>
        </b-form>
      </div>
    </b-modal>

    <router-view :model="model" :secret="secret" :lang="lang"></router-view>
  </div>
</template>

<script>
import NavBar from '@/NavBar.vue'
import ModalDisconnected from "@/maps/modals/ModalDisconnected.vue"

import {MapModel} from 'aktionskarten.js'
import {api} from '@/api.js'

export default {
  name: 'app',
  components: {
    'navbar': NavBar,
    'modal-disconnected': ModalDisconnected,
  },
  props: ['lang'],
  data() {
    return {
      model: null,
      secret: null,
      inputSecretMsg: '',
      inputSecret: '',
      inputSecretState: null,
      showModalDisconnected: false,
    }
  },
  async mounted () {
    console.log("MapBase mounted");
    this.fetchData()
  },
  watch: {
    '$route': 'fetchData',
  },
  methods: {
    async tryLogin(evt) {
      evt.preventDefault();
      if (await this.login(this.inputSecret)) {
        this.inputSecret = ''
        this.inputSecretState = null;
        this.inputSecretMsg = ''
        this.$refs.modalLogin.hide()
      } else {
        console.warn("login failed");
        this.inputSecretMsg = 'Invalid token. Login failed.'
        this.inputSecretState = false;
      }
    },
    logout () {
      this.deleteCookie();
      this.model.logout();
      this.secret = null;
      let params = {id: this.model.id, secret: null, lang: this.lang};
      this.$router.replace({name: this.$route.name, params: params});
    },
    async login (secret) {
      if (secret) {
        if (await this.model.login(secret)) {
          this.secret = secret;
          this.setCookie();
          let params = {id: this.model.id, secret: secret, lang: this.lang};
          this.$router.replace({name: this.$route.name, params: params});
          return true;
        }
      }
      return false;
    },
    async fetchData () {
      // generic error handler
      api.errorHandler = (error) => {
        if (error == 'UNAUTHORIZED') {
          this.inputSecretMsg = 'Your token timed out.'
          this.inputSecret = this.secret;
          this.$refs.modalLogin.show()
        } else {
          console.warn("error", error)
        }
      }

      let id = this.$route.params.id
      let secret = this.$route.params.secret || this.getCookieSecret()
      if (id) {
        this.secret = secret;
        this.model = await MapModel.get(api, id, secret)
      } else {
        this.model = new MapModel(api);
      }

      this.model.on('disconnect', (e) => {
        this.showModalDisconnected = true;
      });

      this.model.on('connect', (e) => {
        this.showModalDisconnected = false;
      });

      this.model.on('authenticated', (e) => {
        let authenticated = e.value
        console.log(authenticated ? 'authenticated' : 'unauthenticated');
        if (!authenticated) {
          let params = Object.assign(this.$route.params, {id: this.model.id, lang: this.lang});
          if ('secret' in params) {
            delete params['secret'];
          }
          this.$router.replace({name: this.$route.name, params: params})
        }
      });

      this.model.on('unauthorized', (e) => {
        this.$refs.modalLogin.show()
      });
    },
    cookieWarning() {
      return confirm(this.$t('dialog.cookies'))
    },
    setCookie() {
      if(this.getCookie() || this.cookieWarning() ) {
        const date = new Date();
        const value = this.$route.params.id + ':' + this.secret
        const years = 5
        date.setTime(date.getTime() + (years*365*24*60*60*1000));
        document.cookie = this.getCookieKey() + '=' + this.secret + ';Path=/;expires=' + date;
      }
    },
    getCookieKey() {
      return 'secret:' + this.$route.params.id;
    },
    getCookie() {
      const splitString = this.getCookieKey();
      const cookies = decodeURIComponent(document.cookie).split(';');
      const cookie = cookies.find(cookie => cookie.indexOf(splitString) == 0);
      return cookie;
    },
    getCookieSecret() {
      const cookie = this.getCookie();
      if (!cookie) return '';
      return cookie.substring(this.getCookieKey().length+1, cookie.length);
    },
    deleteCookie() {
      document.cookie = this.getCookieKey() + '=;Path=/;expires=' + new Date().getUTCDate();
    }
  }
}
</script>
