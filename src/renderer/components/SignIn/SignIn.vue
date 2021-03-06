/*---------------------------------------------------------------------------------------------
 *  Licensed under the GPL-3.0 License. See License.txt in the project root for license information.
 *  You can not delete this comment when you deploy an application.
 *--------------------------------------------------------------------------------------------*/

'use strict';

<template>
  <div>
    <!-- 타이틀바 컴포넌트 -->
    <top-header :data="{ playType: 'list' }"/>
    <div class="wrapper">
      <div class="contents">
        <div class="cover">
          <img width="350" v-if="!isSignin" src="@/assets/images/youtube/dev.png">
          <div v-if="isSignin">
            <div class="picture">
              <img class="userPicture" width="100" :src="profileData.googlePicture">
            </div>
            <div class="userName">{{ profileData.googleName }}</div>
          </div>
        </div>
        <div class="signin" :class="{ signout: isSignin }">
          <el-button
            v-if="!isSignin"
            class="cursor"
            type="primary"
            icon="el-icon-sort"
            @click="signin"
          >{{ $t('SIGN.SIGN_IN') }}</el-button>
          <el-button
            v-else
            class="cursor"
            type="warning"
            icon="el-icon-check"
            @click="signout"
          >{{ $t('SIGN.SIGN_OUT') }}</el-button>
        </div>
        <div class="menu1">
          <label class="gr">
            <i class="el-icon-info"></i>
            <strong>{{ $t('SIGN.NOTICE') }}</strong>
          </label>
        </div>
        <div class="description">
          <strong class="gr" v-if="!isSignin">{{ $t('SIGN.NO_LOGIN_NOTICE') }}</strong>
          <strong class="gr" v-if="isSignin">{{ $t('SIGN.YES_LOGIN_NOTICE') }}</strong>
        </div>
      </div>
    </div>
    <!-- 서브 플레이어 컴포넌트 -->
    <sub-player-bar class="md-top-61" v-show="isMini"/>
  </div>
</template>

<script>
import subPlayerBar from '@/components/PlayerBar/SubPlayerBar'
import storeMixin from '@/components/Mixin/index'
import commonMixin from '@/components/Mixin/common'

export default {
  name: 'SignInPage',
  mixins: [storeMixin, commonMixin],
  components: {
    subPlayerBar
  },
  data () {
    return {
      isSignin: false,
      isMini: false,
      profileData: null,
      playlistCount: 0,
      channelCount: 0
    }
  },
  created () {
    let musicInfo = this.getMusicInfos()
    if (musicInfo) {
      this.isMini = true
    }
    this.$ipcRenderer.on('render:googleAuth', (e, args) => {
      if (args.resultCode === 200) {
        this.$store.commit('setGoogleProfile', JSON.parse(args.body))
        this.success()
      }
    })
  },
  mounted () {
    if (this.getUserId()) {
      this.isSignin = true
      this.profileData = this.getProfile()
    }
  },
  methods: {
    signin () {
      this.$ipcRenderer.send('main:googleAuth', null)
    },
    signout () {
      this.isSignin = false
      this.$store.commit('setGoogleProfile', null)
    },
    getCollectionCount () {
      if (this.getUserId()) {
        this.$local
          .find({
            selector: {
              type: 'profile',
              userId: this.getUserId()
            },
            fields: ['_id', 'collections']
          })
          .then(result => {
            let docs = result.docs[0]
            if (docs) {
              let collections = docs.collections
              this.playlistCount = this.$lodash
                .chain(collections)
                .filter(item => {
                  item.playType === 'play'
                })
                .size()
                .value()

              this.channelCount = this.$lodash
                .chain(collections)
                .filter(item => {
                  item.playType === 'channel'
                })
                .size()
                .value()
            }
          })
          .catch(error => {
            console.log(error)
          })
      }
    },
    success () {
      this.isSignin = true
      this.profileData = this.getProfile()
      if (this.profileData) {
        // 프로필 정보를 받았으면 로그인정보로 프로필이 등록되어있는지 조회한다.
        this.$local
          .find({
            selector: {
              type: 'profile',
              userId: this.getUserId()
            },
            fields: ['_id', 'collections']
          })
          .then(result => {
            // Documents
            let docs = result.docs[0]

            // 프로필이 없으면 새로 프로필을 등록한다
            if (!docs) {
              let data = {
                _id: this.getProfileKey(),
                type: 'profile',
                userId: this.getUserId(),
                history: [],
                playlists: [],
                setting: [],
                collections: [],
                keywords: []
              }
              this.$local
                .post(data)
                .then(res => {
                  this.getCollectionCount()
                })
                .catch(err => {
                  console.log(err)
                })
            } else {
              this.getCollectionCount()
            }
          })
          .catch(err => {
            console.log(err)
          })
      }
    }
  }
}
</script>

<style scoped>
.cover {
  margin-top: 40px;
  text-align: center;
}

.body {
  text-align: center;
  color: #ffffff;
  font-weight: 700;
}

.signin {
  text-align: center;
  margin-top: 20px;
}

.signout {
  text-align: center;
  margin-top: 40px;
}

.gr {
  color: #ffffff;
}

.description {
  margin-left: 30px;
  margin-right: 30px;
  margin-top: 20px;
}

.menu1 {
  margin-top: 70px;
  font-size: 20px;
  margin-left: 30px;
}

.userPicture {
  border-radius: 50px;
}

.userName {
  margin-top: 15px;
  color: #ffffff;
  font-size: 20px;
}
</style>
