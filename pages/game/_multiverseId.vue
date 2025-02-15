<template>
  <div>
    <v-container>
      <div>
        <div class="d-flex justify-center align-center mt-12 mb-6">
          <v-tooltip top open-delay="500">
            <template #activator="{ on }">
              <v-btn class="ml-2" :disabled="!canLock || lockGameLoading" icon v-on="on" @click="toggleGameLock">
                <v-icon :class="multiverse?.locked ? 'lock-animation' : 'unlock-animation'"
                  >{{ multiverse?.locked ? 'mdi-lock' : 'mdi-lock-open-outline' }}
                </v-icon>
              </v-btn>
            </template>
            <span v-if="canLock">{{ multiverse?.locked ? 'Unlock' : 'Lock' }} this game</span>
            <span v-else>This game is {{ multiverse?.locked ? 'locked' : 'unlocked' }}</span>
          </v-tooltip>
          <h1 class="text-center mx-4">Game <small>#</small>{{ multiverseId }}</h1>
          <v-tooltip top open-delay="500">
            <template #activator="{ on }">
              <v-btn icon :disabled="gameLinkCopied" v-on="on" @click="copyGameLink">
                <v-icon>{{ gameLinkCopied ? 'mdi-clipboard-check-outline' : 'mdi-link' }}</v-icon>
              </v-btn>
            </template>
            <span>Copy game link</span>
          </v-tooltip>
        </div>

        <div class="text-center my-6">
          <v-tooltip v-if="!isElectron" bottom>
            <template #activator="{ on }">
              <v-btn color="accent" x-large v-on="on" @click="openInLauncher">
                <v-icon left>mdi-launch</v-icon>
                Open in Launcher
              </v-btn>
            </template>
            <span><kbd>Ctrl</kbd> + Click to close this window</span>
          </v-tooltip>
          <v-tooltip v-else-if="!isSpectating" :disabled="!!ownWorld" bottom>
            <template #activator="{ on }">
              <div v-on="on">
                <v-btn x-large color="accent" :loading="launching" :disabled="!ownWorld" @click="launch()">
                  <img
                    class="launch-icon"
                    :class="{ disabled: !ownWorld }"
                    src="../../assets/images/launch.png"
                    alt=""
                  />
                  Launch
                </v-btn>
              </div>
            </template>
            <span v-if="!multiverseReady || multiverse.universes.length > 0"
              >Create or join a world to launch the game</span
            >
            <span v-else>Create a universe to launch the game</span>
          </v-tooltip>

          <div class="mt-2">
            <v-btn v-if="canEnableRaceMode" text @click="enableRaceModeDialogOpen = true">
              <v-icon left>mdi-timer-play-outline</v-icon>
              Enable race mode
            </v-btn>

            <v-btn v-if="canForfeit" text @click="forfeitDialogOpen = true">
              <v-icon left>mdi-cancel</v-icon>
              Forfeit
            </v-btn>

            <v-btn v-if="canDownloadSpoiler" text @click="downloadSpoilerDialogOpen = true">
              <v-icon left>mdi-eye-outline</v-icon>
              View Spoiler
            </v-btn>
          </div>
        </div>

        <throttled-spinner>
          <div v-if="isLoggedIn && multiverseReady">
            <div class="text-center mb-3">
              <wotw-race-timer
                v-if="isRaceRunning"
                :starting-at="normalGameHandlerState.raceStartingAt"
                :finished-time="normalGameHandlerState.finishedTime ?? null"
              />
              <template v-else-if="isRaceModeEnabled">
                <h3>Waiting for all players to be ready...</h3>
                <div>Start a new save file to signal yourself ready.</div>
              </template>
            </div>

            <div :class="{ 'two-columns': hasRace }">
              <wotw-multiverse-view
                :is-spectating="isSpectating"
                :multiverse="multiverse"
                :player-loading-times="normalGameHandlerState?.playerLoadingTimes"
                :player-finished-times="normalGameHandlerState?.playerFinishedTimes"
                :world-finished-times="normalGameHandlerState?.worldFinishedTimes"
                :universe-finished-times="normalGameHandlerState?.universeFinishedTimes"
              />
              <wotw-race-result-view v-if="hasRace" :race="multiverse.race" />
            </div>

            <div v-if="devtoolsEnabled" class="mt-5">
              <v-card class="pa-4">
                <h3>Dispatch custom event</h3>
                <v-text-field v-model="dev.debugEventName" label="Event" />

                <div class="d-flex">
                  <v-spacer />
                  <v-btn depressed color="accent" @click="dispatchDebugEvent"> Dispatch</v-btn>
                </div>
              </v-card>
            </div>
          </div>
          <div v-if="!isLoggedIn && userLoaded" class="text-center">
            <v-alert class="d-inline-block" color="error darken-3">
              <template v-if="isOBS">
                <b>DO NOT</b> add this page to OBS directly. Please use the "Embed" feature above the board.
              </template>
              <template v-else> You need to be logged in to view this game.</template>
            </v-alert>
          </div>
        </throttled-spinner>
      </div>
    </v-container>

    <template v-if="isLoggedIn && multiverseReady && !!multiverse.bingoBoard">
      <v-container>
        <h2 class="text-center mb-3 mt-8">Bingo board</h2>

        <div class="text-center mb-1">
          <v-btn text @click="centerBoard">
            <template v-if="showBoard">
              <v-icon left>mdi-image-filter-center-focus-strong-outline</v-icon>
              Center on screen
            </template>
            <template v-else>
              <v-icon left>mdi-eye-outline</v-icon>
              Show board
            </template>
          </v-btn>
          <v-btn v-if="isElectron" :disabled="bingoOverlayEnabled" text @click="enableBingoOverlay">
            <template v-if="bingoOverlayEnabled">
              <v-icon left>mdi-check</v-icon>
              Overlay enabled
            </template>
            <template v-else>
              <v-icon left>mdi-semantic-web</v-icon>
              Enable Overlay
            </template>
          </v-btn>
          <v-btn text :disabled="isSpectating" @click="spectateDialogOpen = true">
            <v-icon left>mdi-monitor-eye</v-icon>
            Spectate
          </v-btn>

          <v-menu offset-y :close-on-content-click="!embedUrlLoading && !bingothonTokenLoading">
            <template #activator="{ on, attrs }">
              <v-btn icon v-bind="attrs" class="ml-2" v-on="on">
                <v-icon>mdi-dots-vertical</v-icon>
              </v-btn>
            </template>
            <v-list>
              <v-list-item @click="boardSettingsOpen = true">
                <v-icon left>mdi-cog-outline</v-icon>
                Settings
              </v-list-item>
              <v-list-item :disabled="embedUrlCopied || embedUrlLoading" @click="createEmbedUrl">
                <template v-if="embedUrlCopied">
                  <v-icon left>mdi-check</v-icon>
                  URL Copied
                </template>
                <template v-else>
                  <v-icon left>mdi-semantic-web</v-icon>
                  <template v-if="embedUrlLoading">Generating URL...</template>
                  <template v-else>Embed (OBS)</template>
                </template>
              </v-list-item>
              <v-list-item :disabled="bingothonTokenCopied || bingothonTokenLoading" @click="createBingothonToken">
                <template v-if="bingothonTokenCopied">
                  <v-icon left>mdi-check</v-icon>
                  Token Copied
                </template>
                <template v-else>
                  <v-icon left>mdi-key</v-icon>
                  <template v-if="bingothonTokenLoading">Generating Token...</template>
                  <template v-else>Create Bingothon Token</template>
                </template>
              </v-list-item>
            </v-list>
          </v-menu>
        </div>
      </v-container>

      <div ref="boardContainer" :class="{ 'px-1': !boardSettings.obsMode }" class="board-container">
        <template v-if="showBoard">
          <wotw-bingo-board
            :edge-labels="boardSettings.edgeLabels"
            :is-spectating="isSpectating"
            :multiverse="multiverse"
            :hidden-universes="hiddenUniverses"
            :highlight-universe="highlightedUniverseId"
            :own-universe-id="ownUniverseId"
            :card-attention-effect="boardSettings.cardAttentionEffect"
            :spectator-display-all="boardSettings.spectatorDisplayAll"
            class="board"
          />
          <div class="sidebar px-5">
            <transition-group class="bingo-universes" name="list">
              <div
                v-for="(bingoUniverse, index) in sortedBingoUniverses"
                :key="bingoUniverse.universeId"
                :style="{ zIndex: sortedBingoUniverses.length - index }"
                class="relative"
              >
                <wotw-bingo-universe-view
                  :bingo-universe="bingoUniverse"
                  :is-spectating="isSpectating"
                  :universe="multiverse.universes.find((u) => u.id === bingoUniverse.universeId)"
                  :universe-hidden="hiddenUniverses.includes(bingoUniverse.universeId)"
                  @click="toggleUniverseVisibility(bingoUniverse.universeId)"
                  @click.native.ctrl.capture.stop="toggleUniverseVisibility(bingoUniverse.universeId, true)"
                />
              </div>
              <v-switch
                v-if="isSpectating"
                key="spectatorMode"
                v-model="boardSettings.spectatorDisplayAll"
                label="Show all cards"
                inset
              />
              <div
                v-if="!boardSettings.hideSpectators && multiverse.spectators.length > 0"
                key="spectators"
                class="mt-4"
              >
                <div class="text-caption">Spectators</div>

                <v-tooltip v-for="spectator in multiverse.spectators" :key="spectator.id" top>
                  <span>{{ spectator.name }}</span>
                  <template #activator="{ on }">
                    <span v-on="on">
                      <discord-avatar :user="spectator" />
                    </span>
                  </template>
                </v-tooltip>
              </div>
            </transition-group>
          </div>
        </template>
      </div>

      <v-dialog v-model="boardSettingsOpen" max-width="500">
        <v-card class="pa-5 relative">
          <h2>Bingo settings</h2>

          <v-btn class="close-button" color="background lighten-5" icon @click="boardSettingsOpen = false">
            <v-icon>mdi-close</v-icon>
          </v-btn>

          <v-checkbox
            v-model="boardSettings.edgeLabels"
            hint="Show coordinates around the board"
            label="Edge Labels"
            persistent-hint
          />

          <v-checkbox
            v-model="boardSettings.highlightOwnUniverse"
            hint="Reserve bottom left parts of bingo squares for your team. Greatly improves overview of your progress."
            label="Highlight own team"
            persistent-hint
          />

          <v-checkbox
            v-model="boardSettings.hideSpectators"
            hint="Hide spectators from the player list next to the board"
            label="Hide spectators"
            persistent-hint
          />

          <v-checkbox
            v-model="boardSettings.cardAttentionEffect"
            hint="Flash cards when their value changes"
            label="Flash cards"
            persistent-hint
          />
        </v-card>
      </v-dialog>

      <v-dialog v-model="spectateDialogOpen" :persistent="spectateLoading" max-width="500">
        <v-card class="pa-5 relative">
          <h2>Spectate this game</h2>
          Spectating the game lets you see all squares or the those visible by the teams<br />
          Please note that you <b>cannot join this game anymore</b> after you chose to spectate.

          <div class="d-flex justify-end">
            <v-btn :disabled="spectateLoading" class="mr-1" text @click="spectateDialogOpen = false"> Cancel</v-btn>
            <v-btn :loading="spectateLoading" color="accent" depressed @click="spectate"> Spectate</v-btn>
          </div>
        </v-card>
      </v-dialog>
    </template>

    <v-dialog v-model="enableRaceModeDialogOpen" :persistent="enableRaceModeLoading" max-width="500">
      <v-card class="pa-5 relative">
        <h2>Enable race mode</h2>

        Players will be blocked from starting new games until everyone is ready. To signal yourself ready, select an
        empty save file and choose the difficulty.<br />
        Once the race starts, the game will be locked.

        <div class="d-flex justify-end">
          <v-btn :disabled="enableRaceModeLoading" class="mr-1" text @click="enableRaceModeDialogOpen = false">
            Cancel
          </v-btn>
          <v-btn :loading="enableRaceModeLoading" color="accent" depressed @click="enableRaceMode"
            >Enable race mode
          </v-btn>
        </div>
      </v-card>
    </v-dialog>

    <v-dialog v-model="forfeitDialogOpen" :persistent="forfeitLoading" max-width="400">
      <v-card class="pa-5 relative">
        <h2>Forfeit</h2>

        Forfeiting from the game makes all players in your universe forfeit from this race.

        <div class="d-flex justify-end">
          <v-btn :disabled="forfeitLoading" class="mr-1" text @click="forfeitDialogOpen = false"> Cancel </v-btn>
          <v-btn :loading="forfeitLoading" color="error" depressed @click="forfeit">Forfeit </v-btn>
        </div>
      </v-card>
    </v-dialog>

    <v-dialog v-model="downloadSpoilerDialogOpen" :persistent="downloadSpoilerLoading" max-width="400">
      <v-card class="pa-5 relative">
        <h2>View Spoiler</h2>

        You can view the spoiler for this seed. Note that <b>all players can see that you viewed the spoiler.</b>

        <div class="d-flex justify-end mt-4">
          <v-btn :disabled="downloadSpoilerLoading" class="mr-1" text @click="downloadSpoilerDialogOpen = false">
            Cancel
          </v-btn>
          <v-btn :loading="downloadSpoilerLoading" color="error" depressed @click="downloadSpoiler"
            >Show Spoiler
          </v-btn>
        </div>
      </v-card>
    </v-dialog>

    <v-dialog v-model="viewSpoilerDialogOpen" transition="dialog-bottom-transition" fullscreen>
      <v-sheet v-if="hasSeed">
        <v-toolbar color="secondary">
          <v-toolbar-title>Spoiler</v-toolbar-title>
          <div class="ml-3">
            <v-tooltip v-for="user in multiverse.seedSpoilerDownloadedBy ?? []" :key="user.id" bottom>
              <span>{{ user.name }}</span>
              <template #activator="{ on }">
                <span v-on="on">
                  <discord-avatar :user="user" />
                </span>
              </template>
            </v-tooltip>
          </div>
          <v-spacer />
          <v-btn icon @click="viewSpoilerDialogOpen = false">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-toolbar>
        <div class="spoiler-text pa-3">{{ spoilerText }}</div>
      </v-sheet>
    </v-dialog>
  </div>
</template>

<script>
  import { mapGetters, mapState } from 'vuex'
  import { isElectron } from '~/assets/lib/isElectron'
  import { applyTransparentWindowStyles, isOBS } from '~/assets/lib/obs'
  import {
    MultiverseInfoMessage_GameHandlerType as GameHandlerType,
    NormalGameHandlerState,
  } from '~/assets/proto/messages'
  import { hasOwnProperty } from '~/assets/lib/hasOwnProperty'

  export default {
    name: 'GamePage',
    data: () => ({
      loading: false,
      embedUrlLoading: false,
      embedUrlCopied: false,
      bingothonTokenLoading: false,
      bingothonTokenCopied: false,
      gameLinkCopied: false,
      multiverseReady: false,
      showBoard: false,
      hiddenUniverses: [], // Array of team IDs
      boardSettingsOpen: false,
      boardSettings: {
        edgeLabels: false,
        highlightOwnUniverse: true,
        hideSpectators: false,
        cardAttentionEffect: true,
        spectatorDisplayAll: false,
      },
      spectateDialogOpen: false,
      spectateLoading: false,
      downloadSpoilerDialogOpen: false,
      downloadSpoilerLoading: false,
      viewSpoilerDialogOpen: false,
      spoilerText: '',
      enableRaceModeDialogOpen: false,
      enableRaceModeLoading: false,
      forfeitDialogOpen: false,
      forfeitLoading: false,
      seedgenResultVisible: false,
      hideSeedgenResultCompletely: false,
      bingoOverlayEnabled: false,
      lockGameLoading: false,
      dev: {
        debugEventName: '',
      },
    }),
    computed: {
      ...mapGetters('user', ['isLoggedIn']),
      ...mapState('user', ['user', 'userLoaded']),
      ...mapState('dev', ['devtoolsEnabled']),
      ...mapState('multiverseState', ['multiverses']),
      ...mapState('electron', ['launching']),
      isElectron,
      isOBS() {
        return isOBS()
      },
      multiverseId() {
        return Number(this.$route.params.multiverseId)
      },
      multiverse() {
        return this.multiverses[this.multiverseId]
      },
      sortedBingoUniverses() {
        if (!this.multiverse) {
          return []
        }

        // Only return bingo teams for which we have a team
        const universes = this.multiverse.universes
        return [...this.multiverse.bingoUniverses.filter((b) => universes.some((u) => u.id === b.universeId))].sort(
          (a, b) => {
            const aRank = this.hiddenUniverses.includes(a.universeId) || this.isSpectating ? 0 : a.rank
            const bRank = this.hiddenUniverses.includes(b.universeId) || this.isSpectating ? 0 : b.rank

            const rankDifference = bRank - aRank

            if (rankDifference === 0) {
              return a.universeId - b.universeId
            }

            return rankDifference
          },
        )
      },
      canCreateUniverse() {
        return this.multiverse.universes.length < 8
      },
      ownWorld() {
        if (!this.multiverse) {
          return null
        }

        for (const universe of this.multiverse.universes) {
          const world = universe.worlds.find((world) => world.members.find((player) => player.id === this.user?.id))

          if (world) {
            return world
          }
        }

        return null
      },
      ownUniverse() {
        if (!this.ownWorld) {
          return null
        }

        return this.multiverse.universes.find((universe) =>
          universe.worlds.find((world) => world.id === this.ownWorld?.id),
        )
      },
      ownUniverseId() {
        return this.ownUniverse?.id
      },
      highlightedUniverseId() {
        if (!this.boardSettings.highlightOwnUniverse || this.multiverse.universes.length <= 1) {
          return null
        }

        return this.ownUniverseId
      },
      isSpectating() {
        if (!this.multiverse || !this.user) {
          return false
        }

        return this.multiverse.spectators.some((s) => s.id === this.user.id)
      },
      launcherUrl() {
        return `ori-rando://game/${this.multiverseId}`
      },
      isBingoBoardOverlay() {
        return this.$route.query.isBingoBoardOverlay === 'true'
      },
      isPlayer() {
        return (
          this.multiverse?.universes.some((u) => u.worlds.some((w) => w.members.some((m) => m.id === this.user.id))) ??
          false
        )
      },
      canLock() {
        return this.isPlayer && this.multiverse.isLockable
      },
      normalGameHandlerState() {
        if (!this.multiverseReady || this.multiverse.gameHandlerType !== GameHandlerType.Normal) {
          return null
        }

        return NormalGameHandlerState.decode(this.multiverse.gameHandlerClientInfo)
      },
      isRaceRunning() {
        if (!this.normalGameHandlerState) {
          return false
        }

        return !!this.normalGameHandlerState.raceStartingAt
      },
      isRaceModeEnabled() {
        if (!this.normalGameHandlerState) {
          return false
        }

        return !!this.normalGameHandlerState.raceModeEnabled
      },
      hasRace() {
        return !!this.multiverse?.race
      },
      canEnableRaceMode() {
        if (!this.normalGameHandlerState) {
          return false
        }

        return !this.normalGameHandlerState.raceModeEnabled && this.isPlayer
      },
      ownWorldFinished() {
        if (!this.normalGameHandlerState) {
          return false
        }

        return (
          this.normalGameHandlerState.raceStarted &&
          this.isPlayer &&
          hasOwnProperty(this.normalGameHandlerState.worldFinishedTimes, this.ownWorld.id)
        )
      },
      canForfeit() {
        if (!this.normalGameHandlerState) {
          return false
        }

        return (
          this.normalGameHandlerState.raceStarted &&
          this.isPlayer &&
          !hasOwnProperty(this.normalGameHandlerState.playerFinishedTimes, this.user.id)
        )
      },
      hasSeed() {
        return !!this.multiverse?.seed
      },
      canDownloadSpoiler() {
        return this.hasSeed && (!this.isRaceRunning || this.ownWorldFinished)
      },
    },
    watch: {
      userLoaded: {
        immediate: true,
        async handler(userLoaded) {
          if (userLoaded && !this.isLoggedIn) {
            if (this.$route.query.jwt) {
              await this.$store.dispatch('auth/setJwt', this.$route.query.jwt)
              await this.$store.dispatch('user/updateUser')
            }
          }
        },
      },
      isLoggedIn: {
        immediate: true,
        async handler(isLoggedIn) {
          if (isLoggedIn) {
            await this.$store.dispatch('multiverseState/fetchMultiverse', this.multiverseId)
            await this.$store.dispatch('multiverseState/connectMultiverse', {
              multiverseId: this.multiverseId,
            })
            this.multiverseReady = true
          }
        },
      },
      boardSettings: {
        deep: true,
        handler(boardSettings) {
          window.localStorage.setItem('boardSettings', JSON.stringify(boardSettings))
        },
      },
      // Probably solved?
      // 'multiverse.universes'() { // TO DO: Temporary workaround for orirando/wotw-server#5
      //   if (this.multiverse.bingoBoard) {
      //     this.$store.dispatch('multiverseState/fetchBingoBoard', this.multiverseId)
      //   }
      // },
      multiverseReady: {
        immediate: true,
        handler(multiverseReady) {
          if (multiverseReady && this.multiverse.bingoBoard) {
            this.$nextTick(() => {
              if (isOBS() || this.isBingoBoardOverlay) {
                applyTransparentWindowStyles()
                this.centerBoard()
              }
            })
          }
        },
      },
    },
    mounted() {
      try {
        this.boardSettings = {
          ...this.boardSettings,
          ...JSON.parse(window.localStorage.getItem('boardSettings')),
        }
      } catch (e) {
        console.error('Could not load board settings', e)
      }

      // if (this.seedgenResult) {
      //   this.seedgenResultVisible = true
      //   this.seedgenResultDialogEnabled = true
      //   setTimeout(() => {
      //     this.seedgenResultVisible = false
      //   }, 600)
      // }

      window.addEventListener('scroll', this.onScroll)

      this.$store.dispatch('time/syncTime')
    },
    beforeDestroy() {
      window.removeEventListener('scroll', this.onScroll)
    },
    methods: {
      async join(worldId) {
        await this.$axios.post(`/multiverses/${this.multiverseId}/worlds/${worldId}`)
        await this.$store.dispatch('multiverseState/connectMultiverse', {
          multiverseId: this.multiverseId,
          reconnect: true,
        })
      },
      async createWorld(universeId = null) {
        await this.$axios.post(`/multiverses/${this.multiverseId}/${universeId}/worlds`)
      },
      centerBoard() {
        this.$refs.boardContainer.scrollIntoView({
          behavior: this.showBoard ? 'smooth' : 'auto',
          block: 'start',
        })
        this.showBoard = true
      },
      toggleUniverseVisibility(universeId, exclusive = false) {
        if (exclusive) {
          if (
            this.hiddenUniverses.length === this.sortedBingoUniverses.length - 1 &&
            !this.hiddenUniverses.includes(universeId)
          ) {
            this.hiddenUniverses = []
          } else {
            this.hiddenUniverses = this.sortedBingoUniverses.map((b) => b.universeId).filter((u) => u !== universeId)
          }
        } else if (this.hiddenUniverses.includes(universeId)) {
          this.hiddenUniverses = this.hiddenUniverses.filter((u) => u !== universeId)
        } else {
          this.hiddenUniverses.push(universeId)
        }
      },
      async createEmbedUrl() {
        this.embedUrlLoading = true

        const token = await this.$axios.$post('/tokens/', {
          scopes: ['boards.read', 'user.info.read'],
        })

        const targetRoute = this.$router.resolve({
          name: 'game-multiverseId',
          params: {
            multiverseId: this.multiverseId,
          },
          query: {
            jwt: token,
          },
        })

        const url = new URL(targetRoute.href.replace('#/', ''), this.$paths.UI_BASE_URL)
        await window.navigator.clipboard.writeText(url.toString())

        this.embedUrlLoading = false

        this.embedUrlCopied = true

        setTimeout(() => {
          this.embedUrlCopied = false
        }, 4000)
      },
      async createBingothonToken() {
        this.bingothonTokenLoading = true

        const token = await this.$axios.$post('/bingothon/token')

        await window.navigator.clipboard.writeText(token)

        this.bingothonTokenLoading = false
        this.bingothonTokenCopied = true

        setTimeout(() => {
          this.bingothonTokenCopied = false
        }, 4000)
      },
      async copyGameLink() {
        const url = new URL(`/game/${this.multiverseId}`, this.$paths.UI_BASE_URL)
        await navigator.clipboard.writeText(url.toString())
        this.gameLinkCopied = true

        setTimeout(() => {
          this.gameLinkCopied = false
        }, 3000)
      },
      async spectate() {
        this.spectateLoading = true

        try {
          await this.$store.dispatch('multiverseState/spectateMultiverse', this.multiverseId)
          this.spectateDialogOpen = false
          if (!this.showBoard) {
            this.centerBoard()
          }
        } catch (e) {
          console.error(e)
        }
        this.spectateLoading = false
      },
      onScroll() {
        this.hideSeedgenResultCompletely = document.scrollingElement.scrollTop > 200
      },
      async enableBingoOverlay() {
        await window.electronApi.invoke('bingoBoardOverlay.prepare', this.multiverseId)
        this.bingoOverlayEnabled = true

        setTimeout(() => {
          this.bingoOverlayEnabled = false
        }, 5000)
      },
      openInLauncher(event) {
        window.open(this.launcherUrl, '_self')

        if (event.ctrlKey) {
          setTimeout(() => {
            window.close()
          }, 500)
        }
      },
      async dispatchDebugEvent() {
        try {
          await this.$axios.post(`/multiverses/${this.multiverseId}/debug-event/${this.dev.debugEventName}`)
        } catch (e) {
          console.error(e)
        }
      },
      async dispatchEvent(event) {
        try {
          await this.$axios.post(`/multiverses/${this.multiverseId}/event/${event}`)
        } catch (e) {
          console.error(e)
        }
      },
      async toggleGameLock() {
        if (this.lockGameLoading) {
          return
        }

        this.lockGameLoading = true

        try {
          await this.$axios.post(`/multiverses/${this.multiverseId}/toggle-lock`)
        } catch (e) {
          console.error(e)
        }

        this.lockGameLoading = false
      },
      async launch() {
        if (this.ownWorld?.seedId) {
          const url = `${this.$axios.defaults.baseURL}/world-seeds/${this.ownWorld.seedId}/file`
          const fileName = `${this.ownWorld.seedId}.wotwr`

          await window.electronApi.invoke('launcher.downloadSeedFromUrl', {
            url,
            fileName,
            setToCurrent: true,
            showInExplorer: false,
          })
        }

        await this.$store.dispatch('electron/launch')
      },
      async enableRaceMode() {
        this.enableRaceModeLoading = true
        await this.dispatchEvent('enableRaceMode')
        this.enableRaceModeLoading = false
        this.enableRaceModeDialogOpen = false
      },
      async forfeit() {
        this.forfeitLoading = true
        await this.dispatchEvent('forfeit')
        this.forfeitLoading = false
        this.forfeitDialogOpen = false
      },
      async downloadSpoiler() {
        this.downloadSpoilerLoading = true
        this.spoilerText = await this.$axios.$get(`/seeds/${this.multiverse.seedId}/spoiler`, {
          headers: {
            Accept: 'text/plain',
          },
        })
        this.downloadSpoilerLoading = false
        this.downloadSpoilerDialogOpen = false
        this.viewSpoilerDialogOpen = true
      },
    },
  }
</script>

<style lang="scss" scoped>
  .two-columns {
    display: grid;
    grid-template-columns: 1fr auto;
    gap: 1em;

    @media (max-width: 1264px) {
      grid-template-columns: 100%;
      grid-auto-flow: row;
    }
  }

  .universes {
    align-items: flex-start;
    display: flex;
    flex-wrap: wrap;
    gap: 2em;
    justify-content: center;

    .universe-view {
      min-width: 15vw;
    }
  }

  .board-container {
    display: flex;
    min-height: 100vh;

    .board {
      flex-grow: 0;
      flex-shrink: 0;
      height: 100vh;
      width: 100vh;
    }

    .sidebar {
      align-items: center;
      display: flex;
      flex-direction: column;
      flex-grow: 1;
      justify-content: center;

      .bingo-universes {
        display: flex;
        flex-direction: column;
        gap: 0.3em;
        justify-content: stretch;
      }
    }
  }

  .relative {
    .close-button {
      position: absolute;
      right: 1em;
      top: 1em;
      z-index: 10;
    }
  }

  @keyframes lock-animation {
    0% {
      transform: translateY(0);
    }

    25% {
      transform: translateY(5px);
    }

    100% {
      transform: translateY(0);
    }
  }

  @keyframes unlock-animation {
    0% {
      transform: translateY(0);
    }

    25% {
      transform: translateY(-5px);
    }

    100% {
      transform: translateY(0);
    }
  }

  .lock-animation {
    animation: lock-animation 0.25s forwards ease-out;
  }

  .unlock-animation {
    animation: unlock-animation 0.25s forwards ease-out;
  }

  .launch-icon {
    height: 2.25em;
    width: auto;
    margin-right: 0.5em;
    margin-left: -0.5em;

    &.disabled {
      opacity: 0.4;
      filter: grayscale(1);
    }
  }

  .spoiler-text {
    font-family: Consolas, 'Fira Code', monospace;
    overflow-x: scroll;
    white-space: pre;
  }
</style>