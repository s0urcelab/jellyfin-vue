<template>
  <v-fade-transition>
    <canvas
      v-show="!loading && pixels"
      ref="canvas"
      :key="`canvas-${hash}`"
      :width="width"
      :height="height"
      class="absolute"
    />
  </v-fade-transition>
</template>

<script lang="ts">
import Vue from 'vue';
import greenlet from 'greenlet';
import { decode } from 'blurhash';

const isWasmSupported = ((): boolean => {
  try {
    if (
      typeof WebAssembly === 'object' &&
      typeof WebAssembly.instantiate === 'function'
    ) {
      const module = new WebAssembly.Module(
        Uint8Array.of(0x0, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00)
      );

      if (module instanceof WebAssembly.Module)
        return new WebAssembly.Instance(module) instanceof WebAssembly.Instance;
    }
  } catch (e) {}

  return false;
})();

// eslint-disable-next-line no-console
console.debug(
  isWasmSupported ? 'WebAssembly is supported' : 'WebAssembly is not supported'
);

const getJSPixels = greenlet(
  async (
    hash: string,
    width: number,
    height: number,
    punch: number
    // eslint-disable-next-line require-await
  ): Promise<Uint8ClampedArray> => {
    try {
      return decode(hash, width, height, punch);
    } catch {
      throw new TypeError(`Blurhash ${hash} is not valid`);
    }
  }
);

const getWasmPixels = greenlet(
  async (hash: string, width: number, height: number): Promise<Uint8Array> => {
    const wasmDecode = (await import('blurhash-wasm/blurhash_wasm')).decode;

    try {
      return wasmDecode(hash, width, height) as Uint8Array;
    } catch {
      throw new TypeError(`Blurhash ${hash} is not valid`);
    }
  }
);

/**
 * Decodes blurhash outside the main thread, in a web worker using greenlet,
 * using WASM or native JS implementations for decoding based on browser support.
 *
 * @param {string} hash - Hash to decode.
 * @param {number} width - Width of the decoded pixel array
 * @param {number} height - Height of the decoded pixel array.
 * @param {number} punch - Contrast of the decoded pixels
 * @returns {Uint8ClampedArray|Uint8Array} - Returns the decoded pixels in the proxied response by Comlink
 */
async function getPixels(
  hash: string,
  width: number,
  height: number,
  punch: number
): Promise<Uint8ClampedArray | Uint8Array> {
  try {
    return isWasmSupported
      ? await getWasmPixels(hash, width, height)
      : await getJSPixels(hash, width, height, punch);
  } catch {
    throw new TypeError(`Blurhash ${hash} is not valid`);
  }
}

export default Vue.extend({
  props: {
    hash: {
      type: String,
      required: true
    },
    width: {
      type: Number,
      default: 32
    },
    height: {
      type: Number,
      default: 32
    },
    punch: {
      type: Number,
      default: 1
    }
  },
  data() {
    return {
      loading: true,
      pixels: undefined as Uint8ClampedArray | Uint8Array | undefined
    };
  },
  watch: {
    hash(): void {
      this.$nextTick(() => {
        this.loading = true;
        this.setPixels();
      });
    },
    pixels(): void {
      this.draw();
    }
  },
  mounted() {
    this.setPixels();
  },
  methods: {
    draw(): void {
      const ctx = (this.$refs.canvas as HTMLCanvasElement).getContext('2d');
      const imageData = ctx?.createImageData(this.width, this.height);

      if (imageData && this.pixels) {
        imageData.data.set(this.pixels);
        ctx?.putImageData(imageData, 0, 0);
        this.loading = false;
      }
    },
    async setPixels(): Promise<void> {
      const timeStart = window.performance.now();

      try {
        this.pixels = await getPixels(
          this.hash,
          this.width,
          this.height,
          this.punch
        );
      } catch {
        this.pixels = undefined;
        this.$emit('error');
      }

      console.log(window.performance.now() - timeStart);
    }
  }
});
</script>

<style lang="scss" scoped>
.fade-fast-enter-active,
.fade-fast-leave-active {
  transition: opacity 0.15s;
}

.fade-fast-enter,
.fade-fast-leave-to {
  opacity: 0;
}

.absolute {
  height: 100%;
  width: 100%;
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}
</style>
