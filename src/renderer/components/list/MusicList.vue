/*---------------------------------------------------------------------------------------------
 *  Licensed under the GPL-3.0 License. See License.txt in the project root for license information.
 *  You can not delete this comment when you deploy an application.
 *--------------------------------------------------------------------------------------------*/

'use strict';

<template>
  <div>
    <div id="player">
      <!-- 타이틀바 컴포넌트 -->
      <top-header @reloadMusicList="reload"/>

      <div class="zaudio_wrapper">
        <!-- 커버 영역 -->
        <div class="zaudio_container">
          <div class="side_menu">
            <a class="cursor" @click="goBack">
              <img src="@/assets/images/svg/menu-back.svg" title="Back">
            </a>
            <!-- 컬렉션 등록 -->
            <a class="cursor" v-if="playType !== 'related'" @click="addCollection">
              <like
                ref="likes"
                :isLikeToggle="isLikeToggle"
                :data="data"
                :playType="playType"
                @toggle="toggleChange"
              />
            </a>
          </div>
          <div class>
            <img class="cover" :src="cover">
            <div class="zaudio_trackinfo trackinfo">
              <span
                class="label_channel label_v"
                v-if="playType === 'channel'"
              >{{ $t('COMMONS.LABEL.CHANNEL') }}</span>
              <span
                class="label_playlist label_v"
                v-if="playType === 'play'"
              >{{ $t('COMMONS.LABEL.PLAY_LIST') }}</span>
              <span
                class="label_related label_v"
                v-if="playType === 'related'"
              >{{ $t('COMMONS.LABEL.RELATED') }}</span>
              <br>
              <br>
              <div class="titleflow">
                <span class="zaudio_songtitle">{{ coverTitle.substring(0, 32) }}</span>
                <br>
                <span class="zaudio_songartist">{{ channelTitle }}</span>
                <span class="zaudio_songartist">/ {{ totalTracks }} Tracks</span>
              </div>
            </div>
          </div>
          <div class="overay"></div>
        </div>

        <!-- 목록 영역 -->
        <ul id="list" class="zaudio_playlist" :class="{ dynamicHeight: isMini }">
          <li :id="`item${index}`" v-for="(item, index) in playlist" :key="index">
            <!-- 썸네일 -->
            <img class="thumbnails" :src="item.imageInfo">

            <!-- 비디오 제목 -->
            <span
              class="music-title cursor"
              @click="route(item, index)"
            >{{ item.title.substring(0, 40) }}</span>

            <!-- 비디오 라벨 -->
            <span style="flex-grow:1"></span>
            <span
              class="label_video"
              v-if="item.videoId && item.isLive != 'live'"
            >{{ item.duration }}</span>
            <span class="label_live" v-if="item.videoId && item.isLive == 'live'">LIVE</span>

            <!-- 확장메뉴 컴포넌트 -->
            <context-menu :videoId="item.videoId" :data="item"/>
          </li>
          <!-- 더 보기 -->
          <li v-if="isNext" @click="nextPageLoad">
            <span class="loadMore center cursor" v-if="!isMore">
              <i class="el-icon-refresh load_more"></i>
              {{ $t('COMMONS.MORE') }}
            </span>
            <span class="center" v-if="isMore">
              <i class="el-icon-refresh load_more"></i> LOADING ...
            </span>
          </li>
          <li v-else>
            <span class="end">
              <i class="el-icon-check load_more"></i>
              {{ $t('COMMONS.END') }}
            </span>
          </li>
          <!-- 개발자 가이드라인  -->
          <div class="bottom">
            <img src="@/assets/images/youtube/dev.png">
          </div>
        </ul>
      </div>
    </div>

    <!-- 로딩 컴포넌트 -->
    <transition name="fade">
      <loading v-show="!load"/>
    </transition>

    <!-- 서브 플레이어 -->
    <sub-player-bar v-show="isMini"/>

    <!-- 팝업 컴포넌트 -->
    <v-dialog :width="300" :height="300" :clickToClose="false"/>
  </div>
</template>

<script>
import * as $commons from '@/service/commons-service.js'
import subPlayerBar from '@/components/PlayerBar/SubPlayerBar'
import storeMixin from '@/components/Mixin/index'
import collectionQueryMixin from '@/components/Mixin/collections'
import contextMenu from '@/components/ContextMenu/ContextMenu'
import loading from '@/components/Loader/Loader'
import like from '@/components/Collections/like/like'

export default {
  name: 'MusicList',
  mixins: [storeMixin, collectionQueryMixin],
  components: {
    subPlayerBar,
    loading,
    contextMenu,
    like
  },
  data () {
    return {
      load: false,
      isMini: false,
      isNext: true,
      isMore: false,
      isLikeToggle: false,
      cover: '',
      coverTitle: '',
      channelTitle: '',
      menu: null,
      playType: null,
      selected: null,
      totalTracks: null,
      nextPageToken: null,
      channelPlaylistId: null,
      videoId: null,
      clickIdx: null,
      playlist: [],
      data: null
    }
  },
  created () {
    this.feachData()
  },
  methods: {
    toggleChange (value) {
      this.isLikeToggle = value
    },

    addCollection () {
      if (this.getUserId()) {
        let message = ''
        if (this.isLikeToggle) {
          message = this.$t('COMMONS.COLLECTION.REMOVE')
        } else {
          message = this.$t('COMMONS.COLLECTION.REGISTER')
        }
        this.$modal.show('dialog', {
          title: 'Info',
          text: message,
          buttons: [
            {
              title: 'Yes',
              handler: () => {
                this.$refs.likes.like()
                this.$modal.hide('dialog')
              }
            },
            {
              title: 'Close'
            }
          ]
        })
      } else {
        this.$modal.show('dialog', {
          title: 'Info',
          text: this.$t('COLLECTION.NO_LOGIN'),
          buttons: [
            {
              title: 'Close'
            }
          ]
        })
      }
    },
    reload () {
      this.feachData()
    },
    feachData () {
      let musicInfo = this.getMusicInfos()
      if (musicInfo) {
        this.isMini = true
      }

      let playlistName = null
      this.playType = this.$route.params.playType
      this.playlistId = this.$route.params.id

      let playlistId = this.$route.params.id

      if (this.playType === 'play') {
        playlistName = `PLAYLIST:${playlistId}`
      } else if (this.playType === 'related') {
        playlistName = `RELATED:${playlistId}`
      } else if (this.playType === 'channel') {
        playlistName = `CHANNEL:${playlistId}`
      }

      // -> 여기서 재생목록 체크해야한다.
      let allPlaylist = this.getAllPlayList()

      // 현재 페이지의 재생목록이 존재하는가?
      let findPlaylist = this.$lodash.find(allPlaylist, {
        playlistId: playlistName
      })

      // No
      if (!findPlaylist) {
        let requestURL = ''

        if (this.playType === 'play') {
          requestURL = $commons.youtubePlaylistInfo(playlistId)
        } else if (this.playType === 'related') {
          requestURL = $commons.youtubeVideoResult(playlistId)
        } else if (this.playType === 'channel') {
          requestURL = $commons.youtubeChannelSearch(playlistId)
        }

        this.$http
          .get(requestURL)
          .then(res => {
            let videoInfo = null
            let subChannelId = null

            if (this.playType === 'play') {
              requestURL = $commons.youtubePlaylistItem(playlistId)
            } else if (this.playType === 'related') {
              videoInfo = res.data.items[0]
              requestURL = $commons.youtubeRelatedSearch(playlistId)
            } else if (this.playType === 'channel') {
              subChannelId =
                res.data.items[0].contentDetails.relatedPlaylists.uploads
              requestURL = $commons.youtubePlaylistItem(subChannelId)
            }

            this.$http.get(requestURL).then(res => {
              if (this.$lodash.size(res.data.items) > 0) {
                let pathName = null
                if (this.playType === 'play') {
                  pathName = 'setDuration'
                  this.$store.commit('setMusicList', res.data.items)
                } else if (this.playType === 'related') {
                  pathName = 'setRelatedDuration'
                  res.data.items.unshift(videoInfo)
                  this.$store.commit('setRelatedList', res.data.items)
                } else if (this.playType === 'channel') {
                  pathName = 'setDuration'
                  this.$store.commit('setMusicList', res.data.items)
                }

                this.$store.dispatch(pathName).then(results => {
                  // 만약 갯수가 30가 나누어 참이 아니라면 맨 뒤에 배열을 제거한다.
                  let listSize = this.$lodash.size(results)
                  if (listSize % 30 !== 0) {
                    results.splice(listSize - 1, listSize)
                  }

                  let payload = {
                    playlistName: playlistName,
                    playlistId2: subChannelId || null,
                    nextPageToken: res.data.nextPageToken
                      ? res.data.nextPageToken
                      : null,
                    totalResults: res.data.pageInfo.totalResults,
                    list: results
                  }
                  this.$store.commit('setPlayList', payload)
                  this.getData()
                })
              } else {
                this.errorDialog()
              }
            })
          })
          .catch(error => {
            this.errorDialog()
          })
      } else {
        // Yes

        // 커버 및 재생목록정보를 설정한다.
        this.coverTitle = findPlaylist.list[0].title.substring(0, 35)
        this.channelTitle = findPlaylist.list[0].channelTitle
        this.cover = findPlaylist.list[0].imageInfo
        this.totalTracks = findPlaylist.totalTracks
        this.playlist = findPlaylist.list
        this.nextPageToken = findPlaylist.nextPageToken
        this.channelPlaylistId = findPlaylist.channelPlaylistId
          ? findPlaylist.channelPlaylistId
          : null
        this.isNext = !!findPlaylist.nextPageToken
        this.load = true

        this.data = this.playlist[0]

        // Like 조회
        this.getLike()
      }
    },
    getData () {
      // 재생목록 아이디 (명칭+아이디)
      this.playType = this.$route.params.playType
      let playlistId = this.$route.params.id
      let playlistName = null

      // 모든 재생목록 가져오기
      let allPlaylist = this.getAllPlayList()

      if (this.playType === 'play') {
        playlistName = `PLAYLIST:${playlistId}`
      } else if (this.playType === 'related') {
        playlistName = `RELATED:${playlistId}`
      } else if (this.playType === 'channel') {
        playlistName = `CHANNEL:${playlistId}`
      }

      // 현재 페이지의 재생목록이 존재하는가?
      let findPlaylist = this.$lodash.find(allPlaylist, {
        playlistId: playlistName
      })

      // 채널 재생목록 아이디 (채널 아이디 아님)
      if (this.playType === 'channel') {
        this.channelPlaylistId = findPlaylist.channelPlaylistId
      }

      // 커버설정
      this.coverTitle = findPlaylist.list[0].title.substring(0, 35)
      this.channelTitle = findPlaylist.list[0].channelTitle
      this.cover = findPlaylist.list[0].imageInfo

      // 총 트랙수
      this.totalTracks = findPlaylist.totalTracks

      // 모든 재생목록에서 찾아온 현재 페이지 재생목록
      this.playlist = findPlaylist.list
      this.data = this.playlist[0]

      // 다음 페이지 토큰
      this.nextPageToken = findPlaylist.nextPageToken
        ? findPlaylist.nextPageToken
        : null

      // 토큰이 있으면 true / 없으면 false
      this.isNext = !!findPlaylist.nextPageToken

      // Like 조회
      this.getLike()

      // 1초 후 로딩제거
      setTimeout(() => {
        this.load = true
      }, 1000)
    },
    nextPageLoad () {
      // 재생목록 아이디
      let playlistName = null
      let playlistItem = null

      this.isMore = true
      this.playType = this.$route.params.playType
      let playlistId = this.$route.params.id
      let allPlaylist = this.getAllPlayList()

      // 토큰을 사용한 새 재생목록 가져오기
      if (this.playType === 'play') {
        playlistName = `PLAYLIST:${playlistId}`
        playlistItem = $commons.youtubePagingPlaylistItem(
          playlistId,
          this.nextPageToken
        )
      } else if (this.playType === 'related') {
        playlistName = `RELATED:${playlistId}`
        playlistItem = $commons.youtubePagingRelatedSearch(
          playlistId,
          this.nextPageToken
        )
      } else if (this.playType === 'channel') {
        playlistName = `CHANNEL:${playlistId}`
        playlistItem = $commons.youtubePagingPlaylistItem(
          this.channelPlaylistId,
          this.nextPageToken
        )
      }

      this.$http
        .get(playlistItem)
        .then(res => {
          let pathName = null

          if (this.playType === 'play') {
            pathName = 'setDuration'
            this.$store.commit('setMusicList', res.data.items)
          } else if (this.playType === 'related') {
            pathName = 'setRelatedDuration'
            this.$store.commit('setRelatedList', res.data.items)
          } else if (this.playType === 'channel') {
            pathName = 'setDuration'
            this.$store.commit('setMusicList', res.data.items)
          }

          this.$store.dispatch(pathName).then(results => {
            // 기존 재생목록 뒤로, 토큰으로 조회한 목록을 합친다.
            this.playlist = this.$lodash.concat(this.playlist, results)

            // 토큰여부
            this.nextPageToken = res.data.nextPageToken
              ? res.data.nextPageToken
              : null

            // 토큰이 있으면 true / 없으면 false
            this.isNext = !!this.nextPageToken

            // 전체 재생목록에 등록 된 현재 재생목록에 대한 정보 업데이트
            this.$lodash.forEach(allPlaylist, item => {
              if (item.playlistId === playlistName) {
                let payload = {
                  playlistId: playlistName,
                  appendPlaylist: this.playlist,
                  nextPageToken: this.nextPageToken
                }
                this.$store.commit('setPageAppendList', payload)
              }
            })
            this.isMore = false
          })
        })
        .catch(error => {
          this.errorDialog()
        })
    },
    errorDialog () {
      this.$modal.show('dialog', {
        title: 'Error!',
        text: 'List lookup failed.<br/>Return to search page.',
        buttons: [
          {
            title: 'Go Back',
            handler: () => {
              this.$router.push('/search')
              this.$modal.hide('dialog')
            }
          }
        ]
      })
    },
    errorLogin () {
      this.$modal.show('dialog', {
        title: 'Info',
        text: 'It is available after login.',
        buttons: [
          {
            title: 'Close'
          }
        ]
      })
    },
    goBack () {
      this.$router.push(this.$store.getters.getIndexPath)
    },
    route (items, index) {
      this.$store.commit('setPath', this.$route.path)
      this.$router.push({
        name: 'PLAYING-PLAYLIST',
        params: {
          playType: this.playType,
          id: this.$route.params.id,
          start: index
        }
      })
    }
  }
}
</script>

<style scoped>
.zaudio_wrapper {
  min-height: 516px;
}

.zaudio_playlist {
  max-height: 352px;
  overflow-x: hidden;
  z-index: 100;
}

.cover {
  width: 100%;
  background-position: center;
  max-height: 178px;
  filter: brightness(1.1);
}

.dynamicHeight {
  max-height: 300px;
}

.end {
  margin-left: 115px;
}

.none {
  display: none;
}
</style>
