<template>
  <div id="map" class="w-100 h-100" v-pre></div>
</template>

<script>
import {View} from 'aktionskarten.js'
import {Api, MapModel} from 'aktionskarten.js'
import {api} from '@/api.js'

export default {
  props: ['lang', 'id'],
  data() {
    return {
      model: null,
      view: null,
    }
  },
  watch: {
    '$route': 'render',
    'id': 'render',
  },
  mounted () {
    this.render();
  },
  methods: {
    async render () {
      this.model = await MapModel.get(api, this.id)
      if (!this.model) {
        console.warn("aborting no model");
        return;
      }

      if (!this.view) {
        console.log("initializing view")
        this.view = new View('map', this.model)
        await this.view.init(this.lang);
      }
    }
  }
}
</script>
