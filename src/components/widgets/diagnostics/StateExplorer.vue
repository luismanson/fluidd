<template>
  <json-viewer
    class="py-4"
    :value="state"
    :expand-depth="2"
    :class="$vuetify.theme.isDark ? 'jv-dark' : ''"
    sort
    @keyclick="handleClick"
  />
</template>

<script lang="ts">
import { Component, Mixins } from 'vue-property-decorator'
import StateMixin from '@/mixins/state'
import JsonViewer from 'vue-json-viewer'

@Component({
  components: { JsonViewer }
})
export default class StateExplorer extends Mixins(StateMixin) {
  get state () {
    return {
      printer: this.$store.state.printer.printer
    }
  }

  /*
   * the path vue-json-viewer puts out is completely butchered,
   * we try our best to fix them.
   * e.g. ("printer.tmc2209 extruder" -> "printer['tmc2209 extruder']")
   */
  handleClick (path: string) {
    const sanitizedPath = path
      .replace('$.', '')
      .replace(/\.(\w*[^\w\S.]+\w*)/g, (_, match) => {
        if (isNaN(match)) return `['${match}']`
        return `[${match}]`
      })

    this.$emit('input', sanitizedPath)
  }
}
</script>

<style lang="scss">
.jv-container>.jv-code {
  padding: 0;
}

.jv-container.jv-dark {
  background: transparent;
  color: rgba(255, 255, 255, 0.8);

  .jv-item.jv-object { color: rgba(255, 255, 255, 0.8); }
  .jv-item.jv-array { color: rgba(255, 255, 255, 0.8); }
  .jv-key { color: rgba(255, 255, 255, 0.8); }

  .jv-ellipsis {
    color: rgba(255, 255, 255, 0.5);
    background-color: rgba(255, 255, 255, 0.2);
  }
}
</style>
