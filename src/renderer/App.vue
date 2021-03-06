/*---------------------------------------------------------------------------------------------
 *  Licensed under the GPL-3.0 License. See License.txt in the project root for license information.
 *  You can not delete this comment when you deploy an application.
 *--------------------------------------------------------------------------------------------*/

'use strict';

<template>
  <div id="app">
    <transition name="fade">
      <router-view></router-view>
    </transition>
    <div class="navbar">
      <a class="cursor bdright" @click="route('search')">
        <font-awesome-icon size="sm" icon="search"/>
        {{ $t('MAIN.MENU.SEARCH') }}
      </a>
      <a class="cursor bdright" @click="route('collection')">
        <font-awesome-icon size="sm" icon="headphones-alt"/>
        {{ $t('MAIN.MENU.COLLECTION') }}
      </a>
      <a class="cursor bdright" @click="route('login')">
        <font-awesome-icon size="sm" icon="sign-in-alt"/>
        {{ $t('MAIN.MENU.SIGN') }}
      </a>
      <a class="cursor bdright" @click="route('setting')">
        <font-awesome-icon size="sm" icon="cog"/>
        <el-badge
          is-dot
          class="item"
          style="right:-2px;"
          v-if="isCheck"
        >{{ $t('MAIN.MENU.SETTING') }}</el-badge>
        <span v-else>{{ $t('MAIN.MENU.SETTING') }}</span>
      </a>
    </div>

    <Snow
      v-if="isSnow"
      :active="true"
      zIndex="1000"
      :wind="1"
      :swing="0"
      speed="l"
      color="#ffffff"
    />

    <v-dialog :width="300" :height="300" :clickToClose="false"/>
  </div>
</template>

<script>
import Snow from 'vue-niege'
import storeMixin from '@/components/Mixin/index'

export default {
  name: 'MeTube',
  mixins: [storeMixin],
  components: {
    Snow
  },
  data () {
    return {
      isShow: false,
      isSnow: true,
      isSpinShow: false,
      isCheck: false,
      state: '',
      status: []
    }
  },
  created () {
    // 눈 테마
    this.$eventBus.$on('showSnow', this.showSnow)

    // 비디오 상태 체크 이벤트 수신
    this.$eventBus.$off('statusCheck')
    this.$eventBus.$on('statusCheck', this.videoStatusCheck)

    // 재생 플레이어 상태 체크 이벤트 수신
    this.$eventBus.$on('playerState', this.playerStatusCheck)
  },
  mounted () {
    this.$watch(
      () => {
        return this.state
      },
      (newVal, oldVal) => {
        this.status.push(newVal)
      }
    )
    if (process.env.NODE_ENV === 'production') {
      this.onNewReleaseCheck()
    }
  },
  methods: {
    docs () {
      get(this)
    },
    showSnow (v) {
      this.$set(this, 'isSnow', v)
    },
    route (path) {
      if (path == 'search') {
        this.$router.push({
          name: 'play-search'
        })
      } else if (path == 'collection') {
        this.$router.push({
          name: 'collection'
        })
      } else if (path == 'login') {
        this.$router.push({
          name: 'login'
        })
      } else if (path == 'setting') {
        this.$router.push({
          name: 'setting'
        })
      }
    },
    playerStatusCheck (value) {
      this.state = value
      // 버퍼링 or 일시중지
      if (this.state === 2) {
        // 재생모양 아이콘으로 변경
        this.$eventBus.$emit('playerPlay')
      } else if (this.state === 1) {
        // 일시정지 아이콘으로 변경 (현재 재생 중)
        this.$eventBus.$emit('playerPause')
      } else if (this.state === 0) {
        // 종료일 경우
        // 재생중인 음악정보
        let musicData = this.getMusicInfos()
        let isRepeat = this.getRepeat()

        // 반복여부
        if (isRepeat) {
          this.$ipcRenderer.send('win2Player', [
            'loadVideoById',
            musicData.videoId
          ])
        } else {
          // 이전 음악의 인덱스 (현재 종료된 음악)
          let currentIndex = musicData.index
          // 다음 인덱스
          let nextIndex = currentIndex + 1

          if (musicData.type) {
            this.$local
              .find({
                selector: {
                  type: 'profile',
                  userId: this.getUserId()
                },
                fields: ['playlists']
              })
              .then(result => {
                let docs = result.docs[0]
                let playlists = docs.playlists
                if (playlists) {
                  this.playlist = this.$lodash.find(playlists, {
                    _key: musicData.name
                  })
                  if (this.playlist.tracks.length > nextIndex) {
                    this.$eventBus.$emit('playlist-nextMusicPlay', nextIndex)
                  } else {
                    this.$eventBus.$emit('playlist-nextMusicPlay', 0)
                  }
                }
              })
          } else {
            // 전체 재생 목록
            let allPlaylist = this.getAllPlayList()
            // 재생목록명으로 재생목록 조회
            let playlist = this.$lodash.find(allPlaylist, {
              playlistId: musicData.name
            })

            if (playlist != undefined) {
              if (playlist.list.length > nextIndex) {
                this.$eventBus.$emit('playlist-nextMusicPlay', nextIndex)
              } else {
                // 토큰여부
                let nextPageToken = playlist.nextPageToken
                if (nextPageToken === null) {
                  // 목록의 마지막 번째 음악이 종료되었으므로, 처음부터 재생
                  this.$eventBus.$emit('playlist-nextMusicPlay', 0)
                } else {
                  // 다음 페이지 조회
                  this.$eventBus.$emit('playlist-nextLoad')
                }
              }
            }
          }
        }
      }
    },
    videoStatusCheck () {
      let isTimer = this.$store.getters.getTimer
      if (isTimer) {
        // clear and set
        let isTime = this.$store.getters.getTime
        clearTimeout(isTime)
      }
      this.$store.commit('setTimer', true)
      setTimeout(() => {
        this.$store.commit('setTime', 1000)
        this.statusResult()
      }, 10000)
    },
    statusResult () {
      this.$store.commit('setTimer', false)
      let statusSize = this.$lodash.size(this.status)
      let lastIndex = this.status[statusSize - 1]
      if (lastIndex) {
        if (lastIndex === -1) {
          let musicData = this.getMusicInfos()
          let all = this.getAllPlayList()
          // 다음 인덱스
          let nextIndex = musicData.index + 1
          // 재생목록명으로 재생목록 조회
          let playlist = this.$lodash.find(all, {
            playlistId: musicData.name
          })
          if (playlist != undefined) {
            if (playlist.list.length > nextIndex) {
              this.$eventBus.$emit('playlist-nextMusicPlay', nextIndex)
            }
          }
        }
      }
      this.status = []
    },
    onNewReleaseCheck () {
      this.$db
        .get('adfe10ffbd1f206762f478326800a5b6')
        .then(doc => {
          let live_version = `${doc.version}`
          let local_version = this.$version
          // new version.
          if (live_version != local_version) {
            this.isCheck = true
            this.$store.commit('setVersionCheck', true)
          }
        })
        .catch(err => {
          console.log(err)
        })
    },
    test () {
      this.$modal.show('dialog', {
        title: '알림',
        text: 'Development in progress ...',
        buttons: [
          {
            title: 'Close'
          }
        ]
      })
    }
  }
}
</script>
<style src="./assets/css/zaudio.css"></style>
<style src="./assets/css/commons.css"></style>
<style src="./assets/css/tooltip.css"></style>
<style src="./assets/css/animate.css"></style>
<style src="./assets/css/collection.css"></style>
<style scope>
i {
  padding-right: 5px;
}
.bdright {
  border-right: 1px solid #1b1b1b;
}
/* Place the navbar at the bottom of the page, and make it stick */
.navbar {
  background-color: #111;
  border-top: 1px solid #1b1b1b;
  overflow: hidden;
  position: fixed;
  bottom: 0px;
  width: 100%;
  z-index: 1000;
}
/* Style the links inside the navigation bar */
.navbar a {
  float: left;
  display: block;
  color: #f2f2f2;
  text-align: center;
  padding: 15px 19.4px;
  text-decoration: none;
  font-size: 11px;
  font-weight: 700;
}
/* Change the color of links on hover */
.navbar a:hover {
  background-color: #ffffff;
  color: black;
}
/* Add a color to the active/current link */
.navbar a.active {
  background-color: #4caf50;
  color: white;
}
.position {
  position: absolute;
  bottom: 29px;
  right: 9px;
  width: 14px;
  z-index: 99999;
}
</style>
