<template>
  <div class="d-inline-flex">
    <v-btn
      class="mr-2"
      color="primary"
      min-width="8em"
      depressed
      rounded
      @click="playDirect"
    >
      直接播放
    </v-btn>
  </div>
</template>

<script lang="ts">
import { BaseItemDto } from '@jellyfin/client-axios';
import Vue from 'vue';

export default Vue.extend({
  props: {
    item: {
      type: Object as () => BaseItemDto,
      required: true
    }
  },
  methods: {
    playDirect(): void {
      window.location.href = `directplay://${this.item.Path}`;
    }
  }
});
</script>
