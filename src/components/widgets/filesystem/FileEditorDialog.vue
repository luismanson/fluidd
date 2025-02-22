<template>
  <v-dialog
    :value="value"
    :loading="loading"
    hide-overlay
    fullscreen
    persistent
    transition="dialog-bottom-transition"
    content-class="config-editor-overlay"
    @input="$emit('input', $event)"
  >
    <v-card
      d-flex
      class="fill-height"
      style="overflow: hidden;"
    >
      <v-toolbar
        dense
        :elevation="6"
        style="z-index: 1"
      >
        <app-btn
          v-if="!$vuetify.breakpoint.smAndDown"
          icon
          :disabled="!ready"
          color=""
          @click="emitClose()"
        >
          <v-icon>$close</v-icon>
        </app-btn>
        <v-toolbar-title>{{ filename }}</v-toolbar-title>
        <v-spacer />
        <v-toolbar-items>
          <app-btn
            v-if="!useTextOnlyEditor"
            @click="handleCommandPalette"
          >
            <v-icon
              small
              :left="!$vuetify.breakpoint.smAndDown"
            >
              $consoleLine
            </v-icon>
            <span v-if="!$vuetify.breakpoint.smAndDown">{{ $t('app.file_system.title.command_palette') }}</span>
          </app-btn>
          <app-btn
            v-if="!printerPrinting && configMap.link"
            :href="configMap.link"
            target="_blank"
          >
            <v-icon
              small
              :left="!$vuetify.breakpoint.smAndDown"
            >
              $help
            </v-icon>
            <span v-if="!$vuetify.breakpoint.smAndDown">{{ $t('app.general.btn.config_reference') }}</span>
          </app-btn>
          <app-btn
            v-if="!readonly && !printerPrinting && configMap.serviceSupported"
            :disabled="!ready"
            @click="emitSave(true)"
          >
            <v-icon
              small
              :left="!$vuetify.breakpoint.smAndDown"
            >
              $restart
            </v-icon>
            <span v-if="!$vuetify.breakpoint.smAndDown">{{ $t('app.general.btn.save_restart') }}</span>
          </app-btn>
          <app-btn
            v-if="!readonly"
            :disabled="!ready"
            @click="emitSave(false)"
          >
            <v-icon
              small
              :left="!$vuetify.breakpoint.smAndDown"
            >
              $save
            </v-icon>
            <span v-if="!$vuetify.breakpoint.smAndDown">{{ $t('app.general.btn.save') }}</span>
          </app-btn>
          <app-btn
            @click="emitClose()"
          >
            <v-icon
              small
              :left="!$vuetify.breakpoint.smAndDown"
            >
              $close
            </v-icon>
            <span v-if="!$vuetify.breakpoint.smAndDown">{{ $t('app.general.btn.close') }}</span>
          </app-btn>
        </v-toolbar-items>
      </v-toolbar>

      <file-editor
        v-if="contents !== undefined && !useTextOnlyEditor"
        ref="editor"
        v-model="updatedContent"
        :filename="filename"
        :readonly="readonly"
        :code-lens="codeLens"
        @ready="editorReady = true"
      />

      <file-editor-text-only
        v-if="contents !== undefined && useTextOnlyEditor"
        v-model="updatedContent"
        :filename="filename"
        :readonly="readonly"
        @ready="editorReady = true"
      />
    </v-card>
  </v-dialog>
</template>

<script lang="ts">
import { Component, Mixins, Prop, Ref } from 'vue-property-decorator'
import StateMixin from '@/mixins/state'
import FileEditor from './FileEditor.vue'
import FileEditorTextOnly from './FileEditorTextOnly.vue'
import isMobile from '@/util/is-mobile'
import isWebAssemblySupported from '@/util/is-web-assembly-supported'

@Component({
  components: {
    FileEditor,
    FileEditorTextOnly
  }
})
export default class FileEditorDialog extends Mixins(StateMixin) {
  @Prop({ type: Boolean, required: true })
  readonly value!: boolean

  @Prop({ type: String, required: true })
  readonly root!: string

  @Prop({ type: String, required: true })
  readonly filename!: string

  @Prop({ type: String, required: true })
  readonly contents!: string

  @Prop({ type: Boolean, default: false })
  readonly loading!: boolean

  @Prop({ type: Boolean, default: false })
  readonly readonly!: boolean

  @Ref('editor')
  readonly editor?: FileEditor

  updatedContent = this.contents
  lastSavedContent = this.updatedContent
  editorReady = false

  get ready () {
    return (
      !this.loading &&
      this.editorReady &&
      !this.isUploading
    )
  }

  get isMobile () {
    return isMobile()
  }

  get isWebAssemblySupported () {
    return isWebAssemblySupported()
  }

  get useTextOnlyEditor () {
    return this.isMobile || !this.isWebAssemblySupported
  }

  get isUploading (): boolean {
    return this.$store.state.files.uploads.length > 0
  }

  get rootProperties () {
    return this.$store.getters['files/getRootProperties'](this.root)
  }

  get configMap () {
    return this.$store.getters['server/getConfigMapByFilename'](this.filename)
  }

  get codeLens () {
    return this.$store.state.config.uiSettings.editor.codeLens
  }

  mounted () {
    this.updatedContent = this.contents
    this.lastSavedContent = this.updatedContent
    window.addEventListener('beforeunload', this.handleBeforeUnload)
  }

  beforeDestroy () {
    window.removeEventListener('beforeunload', this.handleBeforeUnload)
  }

  get showDirtyEditorWarning () {
    return this.$store.state.config.uiSettings.editor.confirmDirtyEditorClose && this.updatedContent !== this.lastSavedContent
  }

  async emitClose () {
    if (this.showDirtyEditorWarning) {
      const result = await this.$confirm(
        this.$tc('app.general.simple_form.msg.unsaved_changes'),
        { title: this.$tc('app.general.label.unsaved_changes'), color: 'card-heading', icon: '$error' }
      )

      if (!result) {
        return
      }
    }

    this.$emit('input', false)
  }

  handleBeforeUnload (e: Event) {
    if (this.showDirtyEditorWarning) {
      e.preventDefault() // show browser-native "Leave site?" modal
      return ((e || window.event).returnValue = true) // fallback
    }
  }

  emitSave (restart: boolean) {
    if (this.editorReady) {
      if (this.configMap.serviceSupported && restart) {
        this.$emit('save', this.updatedContent, this.configMap.service)
        this.$emit('input', false)
      } else {
        this.$emit('save', this.updatedContent)
      }

      this.lastSavedContent = this.updatedContent
    }
  }

  handleCommandPalette () {
    this.editor?.showCommandPalette()
  }
}
</script>
