<template>
  <div id="visual-keymap" :style="styles">
    <template v-for="meta in currentLayer">
      <transition name="fade" appear :key="meta.id">
        <component
          v-bind:is="getComponent(meta)"
          v-bind="meta"
          :key="meta.id"
        />
      </transition>
    </template>
  </div>
</template>
<script>
import isUndefined from 'lodash/isUndefined';
import { mapState, mapGetters, mapMutations, mapActions } from 'vuex';
import BaseKeymap from '@/components/BaseKeymap.vue';
import BaseKey from '@/components/BaseKey.vue';
import AnyKey from '@/components/AnyKey.vue';
import LayerKey from '@/components/LayerKey.vue';
import ContainerKey from '@/components/ContainerKey.vue';
import LayerContainerKey from '@/components/LayerContainerKey.vue';

export default {
  name: 'visual-keymap',
  extends: BaseKeymap,
  props: {
    profile: Boolean,
    debug: {
      type: Boolean,
      default: false
    }
  },
  watch: {
    layout(newLayout, oldLayout) {
      // eslint-disable-next-line no-console
      this.profile && console.time('layout');
      if (
        !isUndefined(newLayout) &&
        !this.isLayoutUIUpdate(newLayout, oldLayout) &&
        newLayout !== oldLayout
      ) {
        this.recalcEverything(newLayout);
      } else if (
        isUndefined(newLayout) &&
        !isUndefined(oldLayout) &&
        oldLayout !== ''
      ) {
        this.recalcEverything(oldLayout);
      }
      // eslint-disable-next-line no-console
      this.profile && console.timeEnd('layout');
    }
  },
  computed: {
    ...mapState('keymap', ['config', 'displaySizes', 'layer']),
    ...mapGetters('keymap', [
      'getLayer',
      'loadingKeymapPromise',
      'colorway',
      'defaults'
    ]),
    ...mapState('app', ['layout', 'layouts', 'previewRequested']),
    currentLayer() {
      const layout = this.layouts[this.layout];
      const keymap = this.getLayer(this.layer);
      if (isUndefined(layout) || isUndefined(keymap)) {
        return [];
      }
      if (this.loadingKeymapPromise) {
        const _promise = this.loadingKeymapPromise;
        this.setLoadingKeymapPromise(undefined);
        _promise();
      }
      // Calculate Max with given layout
      // eslint-disable-next-line no-console
      this.profile && console.time('currentLayer');
      const colorway = this.colorway;
      let curLayer = layout.map((pos, index) => {
        let _pos = Object.assign({ w: 1, h: 1 }, pos);
        const coor = this.calcKeyKeymapPos(_pos.x, _pos.y);
        const dims = this.calcKeyKeymapDims(_pos.w, _pos.h);
        return Object.assign(
          {
            id: index,
            layer: this.layer,
            meta: keymap[index],
            colorway: colorway,
            displaySizes: this.displaySizes
          },
          coor,
          dims
        );
      });
      // eslint-disable-next-line no-console
      this.profile && console.timeEnd('currentLayer');
      return curLayer;
    }
  },
  methods: {
    ...mapMutations('keymap', [
      'changeLayer',
      'clear',
      'initKeymap',
      'resetConfig',
      'resizeConfig',
      'setLoadingKeymapPromise'
    ]),
    ...mapMutations('status', ['append']),
    ...mapActions('status', ['scrollToEnd']),
    /**
     * Due to a quirk in how reactivity works we have to clear the layout
     * name to reset the UI to it's old value.
     * We should ignore these events to avoid updating the visual keymap.
     * If either the existing or the new layout is empty string return true.
     * We use a change in layout to decide whether to reset the keymap.
     */
    isLayoutUIUpdate(newLayout, oldLayout) {
      return newLayout === '' || oldLayout === '';
    },
    getComponent(key) {
      const { meta } = key;
      if (meta === undefined) {
        this.debug && console.log(`key ${key.id} has undefined metadata`);
        return BaseKey;
      }
      switch (meta.type) {
        case 'container':
          return ContainerKey;
        case 'layer':
          return LayerKey;
        case 'layer-container':
          return LayerContainerKey;
        case 'text':
          return AnyKey;
        default:
          return BaseKey;
      }
    },
    setSize(max) {
      this.width = max.x;
      this.height = max.y;
    },
    recalcEverything(newLayout) {
      // eslint-disable-next-line no-console
      this.profile && console.time('layout::reset');
      this.resetConfig();
      this.changeLayer(0);
      // eslint-disable-next-line no-console
      this.profile && console.time('layout::initkeymap');
      if (this.getLayer(0).length === 0) {
        this.initKeymap({
          layer: 0,
          layout: this.layouts[newLayout]
        });
      }
      // eslint-disable-next-line no-console
      this.profile && console.timeEnd('layout::initkeymap');
      // eslint-disable-next-line no-console
      this.profile && console.timeEnd('layout::reset');

      // eslint-disable-next-line no-console
      this.profile && console.time('layout::scale');
      const layout = this.layouts[newLayout];
      if (isUndefined(layout)) {
        const msg = `\n\nWARNING: layout ${newLayout} does not exist on this keyboard\n\n`;
        console.log(msg);
        this.append(msg);
        this.scrollToEnd();
        return;
      }
      const max = layout.reduce(
        (acc, pos) => {
          let _pos = Object.assign({ w: 1, h: 1 }, pos);
          const coor = this.calcKeyKeymapPos(_pos.x, _pos.y);
          const dims = this.calcKeyKeymapDims(_pos.w, _pos.h);
          acc.x = Math.max(acc.x, coor.x + dims.w);
          acc.y = Math.max(acc.y, coor.y + dims.h);
          return acc;
        },
        {
          x: 0,
          y: 0
        }
      );
      if (max.x > this.defaults.MAX_X) {
        this.resizeConfig(max);
        max.x *= this.config.SCALE;
        max.y *= this.config.SCALE;
      }
      this.setSize(max);
      // eslint-disable-next-line no-console
      this.profile && console.timeEnd('layout::scale');
    }
  },
  data() {
    return {
      width: 0,
      height: 0
    };
  },
  components: { BaseKey, AnyKey, LayerKey, ContainerKey }
};
</script>
<style>
.fade-enter-active {
  transition: all 0.5s ease;
}
.fade-leave-active {
  transition: all 0.8s cubic-bezier(1, 0.5, 0.8, 1);
}
.fade-enter,
.fade-leave-to {
  opacity: 0;
}
</style>
