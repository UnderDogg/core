<template>
  <ul ref="menu" class="menu song-menu" v-show="shown" tabindex="-1" @contextmenu.prevent @blur="close" :style="{ top: `${top}px`, left: `${left}px` }"
  >
    <template v-show="onlyOneSongSelected">
      <li class="playback" @click.stop.prevent="doPlayback">
        <span v-if="firstSongPlaying">Pause</span>
        <span v-else>Play</span>
      </li>
      <li class="go-to-album" @click="viewAlbumDetails(songs[0].album)">Go to Album</li>
      <li class="go-to-artist" @click="viewArtistDetails(songs[0].artist)">Go to Artist</li>
    </template>
    <li class="has-sub">Add To
      <ul class="menu submenu menu-add-to">
        <li class="after-current" @click="queueSongsAfterCurrent">After Current Song</li>
        <li class="bottom-queue" @click="queueSongsToBottom">Bottom of Queue</li>
        <li class="top-queue" @click="queueSongsToTop">Top of Queue</li>
        <li class="separator"></li>
        <li class="favorite" @click="addSongsToFavorite">Favorites</li>
        <li class="separator" v-if="playlistState.playlists.length"></li>
        <li
          class="playlist"
          v-for="p in playlistState.playlists"
          :key="p.id"
          @click="addSongsToExistingPlaylist(p)">{{ p.name }}</li>
      </ul>
    </li>
    <li class="open-edit-form" v-if="isAdmin" @click="openEditForm">Edit</li>
    <li class="download" v-if="sharedState.allowDownload" @click="download">Download</li>
    <li class="copy-url" v-if="copyable && onlyOneSongSelected" @click="copyUrl">Copy Shareable URL</li>
  </ul>
</template>

<script>
import { each } from 'lodash'

import songMenuMethods from '@/mixins/song-menu-methods'
import { event, isClipboardSupported, copyText } from '@/utils'
import { sharedStore, songStore, queueStore, userStore, playlistStore } from '@/stores'
import { playback, download } from '@/services'
import router from '@/router'

export default {
  name: 'song-menu',
  props: {
    songs: {
      type: Array,
      required: true
    }
  },
  mixins: [songMenuMethods],

  data () {
    return {
      playlistState: playlistStore.state,
      sharedState: sharedStore.state,
      copyable: isClipboardSupported()
    }
  },

  computed: {
    onlyOneSongSelected () {
      return this.songs.length === 1
    },

    firstSongPlaying () {
      return this.songs[0] ? this.songs[0].playbackState === 'playing' : false
    },

    isAdmin () {
      return userStore.current.is_admin
    }
  },

  methods: {
    open (top = 0, left = 0) {
      if (!this.songs.length) {
        return
      }

      this.top = top
      this.left = left
      this.shown = true

      this.$nextTick(() => {
        // Make sure the menu isn't off-screen
        if (this.$el.getBoundingClientRect().bottom > window.innerHeight) {
          this.$el.style.top = 'auto'
          this.$el.style.bottom = 0
        } else {
          this.$el.style.top = this.top
          this.$el.style.bottom = 'auto'
        }

        this.$refs.menu.focus()
      })
    },

    /**
     * Take the right playback action based on the current playback state.
     */
    doPlayback () {
      switch (this.songs[0].playbackState) {
        case 'playing':
          playback.pause()
          break
        case 'paused':
          playback.resume()
          break
        default:
          queueStore.contains(this.songs[0]) || queueStore.queueAfterCurrent(this.songs[0])
          playback.play(this.songs[0])
          break
      }

      this.close()
    },

    /**
     * Trigger opening the "Edit Song" form/overlay.
     */
    openEditForm () {
      this.songs.length && event.emit(event.$names.EDIT_SONGS, this.songs)
      this.close()
    },

    /**
     * Load the album details screen.
     */
    viewAlbumDetails (album) {
      router.go(`album/${album.id}`)
      this.close()
    },

    /**
     * Load the artist details screen.
     */
    viewArtistDetails (artist) {
      router.go(`artist/${artist.id}`)
      this.close()
    },

    download () {
      download.fromSongs(this.songs)
      this.close()
    },

    copyUrl () {
      copyText(songStore.getShareableUrl(this.songs[0]))
    }
  },

  /**
   * On component mounted(), we use some JavaScript to prepare the submenu triggering.
   * With this, we can catch when the submenus shown or hidden, and can make sure
   * they don't appear off-screen.
   */
  mounted () {
    each(Array.from(this.$el.querySelectorAll('.has-sub')), item => {
      const submenu = item.querySelector('.submenu')
      if (!submenu) {
        return
      }

      item.addEventListener('mouseenter', e => {
        submenu.style.display = 'block'

        // Make sure the submenu isn't off-screen
        if (submenu.getBoundingClientRect().bottom > window.innerHeight) {
          submenu.style.top = 'auto'
          submenu.style.bottom = 0
        }
      })

      item.addEventListener('mouseleave', e => {
        submenu.style.top = 0
        submenu.style.bottom = 'auto'
        submenu.style.display = 'none'
      })
    })
  }
}
</script>

<style lang="scss" scoped>
@import "~#/partials/_vars.scss";
@import "~#/partials/_mixins.scss";

.menu {
  @include context-menu();
  position: fixed;

  li {
    position: relative;
    padding: 4px 12px;
    cursor: default;
    white-space: nowrap;

    &:hover {
      background: $colorOrange;
      color: #fff;
    }

    &.separator {
      pointer-event: none;
      padding: 1px 0;
      background: #ccc;
    }

    &.has-sub {
      padding-right: 24px;

      &:after {
        position: absolute;
        right: 12px;
        top: 4px;
        content: "▸";
        width: 16px;
        text-align: right;
      }
    }
  }

  .submenu {
    position: absolute;
    display: none;
    left: 100%;
    top: 0;
  }
}
</style>
