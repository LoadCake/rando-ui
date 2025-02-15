<template>
  <div class="d-flex align-center">
    <v-tabs v-if="displayedWorldCount >= 2" v-model="model" background-color="transparent">
      <v-tab
        v-for="(worldPreset, index) in universePreset.worldSettings"
        :key="index"
        :disabled="disabled"
        @contextmenu="event => openWorldContextMenu(event, index)"
      >
        <v-icon left>mdi-earth</v-icon>
        {{ index + 1 }}
      </v-tab>

      <v-tab v-if="addingNewWorld" key="new-world" :disabled="disabled">
        <v-icon left>mdi-earth</v-icon>
        new
      </v-tab>

      <div v-if="!addingNewWorld" class="d-flex align-center pl-3">
        <v-btn :disabled="disabled" icon @click="$emit('add-world')">
          <v-icon>mdi-plus</v-icon>
        </v-btn>
      </div>
    </v-tabs>
    <v-btn
      v-else
      class="my-1"
      :disabled="disabled || universePreset.worldSettings.length === 0"
      text
      @click="$emit('add-world')"
    >
      <v-icon left>mdi-plus</v-icon>
      Multiworld
    </v-btn>

    <v-spacer />

    <v-menu v-model="worldMenuOpen" :position-x="worldMenuX" :position-y="worldMenuY" absolute offset-y>
      <v-list>
        <v-list-item :disabled="disabled" @click="$emit('copy-world', worldMenuWorldIndex)">
          <v-icon left>mdi-content-duplicate</v-icon>
          Copy world
        </v-list-item>
        <v-list-item :disabled="disabled" @click="$emit('delete-world', worldMenuWorldIndex)">
          <v-icon left>mdi-delete-outline</v-icon>
          Delete world
        </v-list-item>
      </v-list>
    </v-menu>

    <v-menu offset-y left close-on-content-click :disabled="disabled">
      <template #activator="{ on, attrs }">
        <v-btn icon v-bind="attrs" class="ml-2" :disabled="disabled" v-on="on">
          <v-icon>mdi-dots-vertical</v-icon>
        </v-btn>
      </template>
      <v-list>
        <v-list-item :disabled="disabled" @click="$emit('start-over')">
          <v-icon left>mdi-delete-outline</v-icon>
          Start over
        </v-list-item>
        <v-list-item :disabled="disabled" @click="$emit('restore-last-config')">
          <v-icon left>mdi-restore</v-icon>
          Restore last config
        </v-list-item>
      </v-list>
    </v-menu>
  </div>
</template>

<script>
  import { hasModelObject } from '~/assets/lib/hasModelObject'

  export default {
    name: 'WotwSeedgenToolbar',
    mixins: [hasModelObject],
    props: {
      universePreset: {
        type: Object,
        required: true,
      },
      disabled: {
        type: Boolean,
        default: false,
      },
      addingNewWorld: {
        type: Boolean,
        default: false,
      },
    },
    data: () => ({
      worldMenuWorldIndex: null,
      worldMenuOpen: false,
      worldMenuX: 0,
      worldMenuY: 0,
    }),
    computed: {
      displayedWorldCount() {
        return this.universePreset.worldSettings.length + (this.addingNewWorld ? 1 : 0)
      },
    },
    methods: {
      openWorldContextMenu(event, worldMenuWorldIndex) {
        this.worldMenuWorldIndex = worldMenuWorldIndex
        event.preventDefault()
        this.worldMenuOpen = false
        this.worldMenuX = event.clientX
        this.worldMenuY = event.clientY
        this.$nextTick(() => {
          this.worldMenuOpen = true
        })
      },
    },
  }
</script>

<style lang="scss" scoped></style>
