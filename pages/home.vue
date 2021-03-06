<template>
  <section class="home-wrapper">
    <v-app id="inspire">
      <v-navigation-drawer
        v-model="drawer"
        app
        clipped
      >
        <v-list dense>
          <v-list-item-group v-model="activeMenu">
            <v-list-item
              :key="item.text"
              v-for="item in menus"
              link
            >
              <v-list-item-action>
                <v-icon>{{ item.icon }}</v-icon>
              </v-list-item-action>
              <v-list-item-content>
                <v-list-item-title>
                  {{ item.text }}
                </v-list-item-title>
              </v-list-item-content>
            </v-list-item>
          </v-list-item-group>
          <v-autocomplete
            v-if="rooms.length > 0"
            search-input="search"
            loading="isLoading"
            hide-no-data
            prepend-inner-icon="mdi-magnify"
            class="mx-5 my-2"
            label="SEARCH"
            clearable
            rounded
            dense
            outlined
          />
          <v-expansion-panels class="mx-0 my-0" :value="0" :flat="true" :hover="true">
            <v-expansion-panel>
              <v-expansion-panel-header v-if="rooms.length > 0" class="my-0 grey--text text--darken-1">
                ROOMS
              </v-expansion-panel-header>
              <v-expansion-panel-content class="expand-content">
                <Rooms :active="activeRooms" @roomSelected="handleActiveRoomChange" :rooms="rooms" />
              </v-expansion-panel-content>
            </v-expansion-panel>
          </v-expansion-panels>
          <v-expansion-panels :value="0" :flat="true" :hover="true">
            <v-expansion-panel>
              <v-expansion-panel-header v-if="activeRooms.length > 0" class="mt-4 grey--text text--darken-1">
                MEMBERS
              </v-expansion-panel-header>
              <v-expansion-panel-content class="expand-content">
                <Members v-if="activeRooms.length > 0" :users="users"/>
              </v-expansion-panel-content>
            </v-expansion-panel>
          </v-expansion-panels>
          <v-menu
            :close-on-content-click="false"
            :nudge-width="200"
            v-model="roomCreatorVisible"
            transition="scale-transition"
            offset-x
          >
            <template v-slot:activator="{ on }">
              <v-list-item
                v-on="on"
                class="mt-4"
                link
              >
                <v-list-item-action>
                  <v-icon color="grey darken-1">
                    mdi-plus-circle-outline
                  </v-icon>
                </v-list-item-action>
                <v-list-item-title class="grey--text text--darken-1">
                  Create Room
                </v-list-item-title>
              </v-list-item>
            </template>
            <v-card @keydown.enter="createRoom">
              <v-list-item three-line>
                <v-list-item-content>
                  <div class="overline mb-4">
                    NEW ROOM
                  </div>
                  <v-text-field
                    ref="input"
                    :autofocus="true"
                    :error-messages="errorMessages"
                    :success-messages="successMessages"
                    :loading="loading"
                    v-model="roomName"
                    required
                    placeholder="Please input your room name"
                  />
                </v-list-item-content>
              </v-list-item>
              <v-card-actions>
                <v-spacer />
                <v-btn @click="roomCreatorVisible = false" text>
                  Cancel
                </v-btn>
                <v-btn @click="createRoom" :disabled="successMessages.length === 0" color="primary" text>
                  Create
                </v-btn>
              </v-card-actions>
            </v-card>
          </v-menu>
          <v-list-item link>
            <v-list-item-action>
              <v-icon color="grey darken-1">
                mdi-settings
              </v-icon>
            </v-list-item-action>
            <v-list-item-title class="grey--text text--darken-1">
              Manage Subscriptions
            </v-list-item-title>
          </v-list-item>
        </v-list>
      </v-navigation-drawer>

      <v-app-bar
        app
        clipped-left
        color="teal"
        dense
      >
        <v-app-bar-nav-icon @click.stop="drawer = !drawer" />
        <v-icon class="mx-4">
          mdi-incognito
        </v-icon>
        <v-toolbar-title class="mr-12 align-center">
          <span class="title">Keyboard-Hero</span>
        </v-toolbar-title>
        <v-spacer />
      </v-app-bar>

      <v-content>
        <Chat
          v-if="activeRooms.length >= 1"
          @leaveRoom="handleLeaveRoom"
          @notify="notifyHandler"
          :username="username"
          :room="openedRoom"
          :messages="currentMessages"
        />
      </v-content>
    </v-app>
    <v-snackbar
      v-model="snackbarProps.value"
      v-bind="snackbarProps"
    >
      {{ snackbarProps.content }}
      <v-btn
        @click="$set(snackbarProps, 'value', false)"
        dark
        text
      >
        Close
      </v-btn>
    </v-snackbar>
  </section>
</template>

<script>
import { debounce } from 'lodash'
import Members from '../components/Members'

import Rooms from '../components/Rooms'
import Chat from '../components/Chat'
import socket from '../plugins/socket.io'

export default {
  components: {
    Members,
    Rooms,
    Chat
  },
  data () {
    return {
      roomName: '',
      roomCreatorVisible: false,
      drawer: null,
      activeMenu: 0,
      activeRooms: [],
      errorMessages: [],
      successMessages: [],
      isLoading: false,
      loading: false,
      snackbarProps: {
        value: false,
        top: 'top',
        color: 'error',
        timeout: 2000,
        content: 'DEFAULT CONTENT'
      },
      messages: {},
      menus: [
        { icon: 'mdi-message', text: 'Messages' },
        { icon: 'mdi-trending-up', text: 'Trending' },
        { icon: 'mdi-history', text: 'History' },
        { icon: 'mdi-playlist-music', text: 'Playlists' }
      ],
      rooms: []
    }
  },
  computed: {
    openedRoom () {
      const INDEX = this.rooms.findIndex(item => item.name === this.activeRooms[0])
      return this.rooms[INDEX]
    },
    currentMessages () {
      return this.messages[this.activeRooms[0]]
    },
    username () {
      const TMP = sessionStorage.getItem('username')
      if (TMP !== null) {
        return TMP
      }
      return socket.id
    },
    users () {
      const ROOM = this.rooms.find(item => item.name === this.activeRooms[0]) || { members: [] }
      return ROOM.members
    }
  },
  watch: {
    roomName (value) {
      if (value === '') {
        this.successMessages = []
        this.errorMessages = []
        return
      }

      this.successMessages.splice(0, this.successMessages.length)
      this.errorMessages.splice(0, this.errorMessages.length)

      if (value.length > 10) {
        this.errorMessages.push('Room name must be less than 16 characters')
        this.successMessages.splice(0, this.successMessages.length)
      } else {
        this.loading = true
        this.checkName(value)
      }
    }
  },
  created () {
    this.$vuetify.theme.dark = true
  },
  beforeDestroy () {
    socket.off('rooms')
    socket.off('rooms::update')
    socket.off('message')
  },
  mounted () {
    const USERNAME = sessionStorage.getItem('username')
    if (USERNAME) {
      socket.emit('submitUsername', USERNAME)
    }
    socket.emit('getRooms')
    socket.on('rooms', (data) => {
      this.rooms.splice(0, this.rooms.length, ...data)
    })
    socket.on('message', ({ room, username, content, action }) => {
      const MSG_ENTRY = { username, content, action }

      if (this.messages[room] === undefined) {
        this.$set(this.messages, room, [MSG_ENTRY])
      } else {
        this.messages[room].push(MSG_ENTRY)
      }
    })
    socket.on('rooms::update', (room) => {
      const INDEX = this.rooms.findIndex(item => item.name === room.name)
      if (INDEX === -1) {
        socket.emit('getRooms')
      } else {
        this.$set(this.rooms, INDEX, room)
      }
    })
  },
  methods: {
    search () {},
    handleLeaveRoom (roomName) {
      const INDEX = this.activeRooms.indexOf(roomName)
      if (INDEX > -1) {
        this.activeRooms.splice(INDEX, 1)
      }
    },
    notifyHandler (msg) {
      this.snackbarProps = Object.assign({}, this.snackbarProps, msg, { value: true })
    },
    handleActiveRoomChange (val) {
      const index = val === undefined ? 0 : this.activeRooms.indexOf(this.rooms[val].name)
      if (index === 0) {
        return
      }
      if (index !== -1) {
        this.activeRooms.splice(index, 1)
        this.activeRooms === [] ? this.activeRooms.push(this.rooms[val].name) : this.activeRooms.unshift(this.rooms[val].name)
      } else {
        socket.emit('join', this.rooms[val].name, this.username)
        this.activeRooms === [] ? this.activeRooms.push(this.rooms[val].name) : this.activeRooms.unshift(this.rooms[val].name)
      }
    },
    checkName: debounce(function (value) {
      return this.$axios.get(window.location.origin + '/check_room', { params: { name: value } }).then(
        ({ data }) => {
          if (data.creatable) {
            this.successMessages.push('room name is invalid')
          } else {
            this.errorMessages.push('room name is invalid')
          }
          this.loading = false
        }
      )
    }, 500, {
      'leading': false,
      'trailing': true
    }),
    createRoom () {
      socket.emit('join', this.roomName)
      this.activeRooms.push(this.roomName)
      this.roomName = ''
      this.roomCreatorVisible = false
    }

  }
}
</script>
<style scoped lang="scss">
  .expand-content {
    max-height: 30vh;
    overflow: scroll;
    overflow: -moz-scrollbars-none;
    -ms-overflow-style: none;
    &::-webkit-scrollbar {
      display: none !important;
      width: 0 !important;
    }
  }
</style>
