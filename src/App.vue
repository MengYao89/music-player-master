<template>
  <div class="app" id="app">
    <div :style="backgroundObj" class="backImg"></div>
    <div class="mask"></div>
    <div class="main-content">
      <div class="search-container">
        <div class="search-wrapper">
          <input v-model="searchInput" type="text" class="search" placeholder="搜索关键字" @keyup.enter="searchKeyword">
          <button class="search-btn" @click="searchKeyword">
            <i class="search-icon"></i>
          </button>
        </div>
      </div>
      <div class="list-wrapper" id="listWrapper">
        <music-list :songList='songList' :currIndex="index" :currentSong="currentSong" :playing="playing" :page="page" :allNum="allNum" :loading="listLoading" @playControl="playControl" @loadMore="loadMore"></music-list>
      </div>
      <div class="lyrics-wrapper">
        <lyrics ref="lyrics" :song="currentSong"></lyrics>
      </div>
    </div>
    <div class="player-wrapper">
      <music-player :playMode="playMode" :playing="playing" :song="currentSong" @nextSong="nextSong" @prevSong="prevSong" @switchState="switchState" @changeMode="changeMode" @renderLyrics="renderLyrics" @checkLyric="checkLyric"></music-player>
    </div>
  </div>
</template>

<script>
import MusicPlayer from './components/MusicPlayer'
import MusicList from './components/MusicList'
import Lyrics from './components/Lyrics'
import IScroll from 'iscroll'
export default {
  name: 'app',
  components: {
    MusicPlayer,
    MusicList,
    Lyrics
  },
  data () {
    return {
      urlSearch: 'https://route.showapi.com/213-1?showapi_appid=42915&showapi_sign=e5c70803e5744a2aae12df378acf3ed0&keyword=',
      searchText: '苏打绿',
      searchInput: '',
      songList: [],
      playedList: [],
      currentSong: {},
      index: 0,
      page: 1,
      allNum: 1,
      playing: false,
      playMode: 'order',
      listScroll: null,
      scrollPos: 0,
      listLoading: false
    }
  },
  methods: {
    default () {
      this.search((data) => {
        this.updateSongData(data)
        this.currentSong = this.songList[this.index]
        this.$nextTick(() => {
          this.initScroll()
        })
      })
    },
    resetList () {
      this.page = 1
      this.index = 0
      this.playedList = []
    },
    searchKeyword () {
      let text = this.searchInput
      if (text.trim() === '') {
        return
      }
      this.songList = []
      this.listLoading = true

      let array = text.split(' ')
      text = ''
      for (let i = 0; i < array.length; i++) {
        if (array[i] === '') {
          array.splice(i, 1)
          --i
          continue
        }
        text += array[i] + '+'
      }
      this.searchText = text
      this.search((data) => {
        this.resetSongData(data)
        this.resetList()
        this.currentSong = this.songList[this.index]
        this.$nextTick(() => {
          this.reinitScroll()
        })
      })
    },
    loadMore () {
      if (this.songList.length >= this.allNum) {
        return
      }
      ++this.page
      this.search((data) => {
        this.updateSongData(data)
        this.$nextTick(() => {
          this.reinitScroll()
        })
      })
    },
    search (callback) {
      this.listLoading = true
      this.$http.get(this.urlSearch + this.searchText + '&page=' + this.page).then((response) => {
        let data = response.data.showapi_res_body.pagebean
        callback.call(this, data)
      })
    },
    resetSongData (data) {
      this.listLoading = false
      this.page = data.currentPage
      this.allNum = data.allNum
      this.songList = data.contentlist
    },
    updateSongData (data) {
      this.listLoading = false
      this.page = data.currentPage
      this.allNum = data.allNum
      this.songList = this.songList.concat(data.contentlist)
    },
    initScroll () {
      if (this.listScroll === null) {
        this.listScroll = new IScroll('#listWrapper', {
          mouseWheel: true,
          scrollbars: 'custom',
          startY: this.scrollPos
        })
      }
    },
    reinitScroll () {
      this.scrollPos = this.listScroll.y
      if (this.listScroll !== null) {
        this.listScroll.destroy()
        this.listScroll = null
      }
      this.initScroll()
    },
    addPlayedList () {
      let data = {
        song: this.currentSong,
        index: this.index
      }
      this.playedList.push(data)
    },
    playControl (index) {
      if (this.songList[index].songid === this.currentSong.songid) {
        this.switchState()
        return
      }
      this.addPlayedList()
      this.index = index
      this.playing = true
      this.currentSong = this.songList[index]
    },
    prevSong () {
      if (this.playedList.length !== 0) {
        let data = this.playedList.pop()
        this.currentSong = data.song
        this.index = data.index
        return
      }
      this.prevHandler()
    },
    nextSong () {
      this.addPlayedList()
      this.nextHandler()
    },
    prevHandler () {
      let t = {
        'order': () => {
          if (this.index === 0) {
            this.index = this.songList.length - 1
          } else {
            --this.index
          }
        },
        'random': () => {
          this.index = Math.floor(this.songList.length * Math.random())
        },
        'cycle': () => {
          if (this.index === 0) {
            this.index = this.songList.length - 1
          } else {
            --this.index
          }
        }
      }
      t[this.playMode].call(this)
      this.currentSong = this.songList[this.index]
    },
    nextHandler () {
      let t = {
        'order': () => {
          if (this.index === this.songList.length - 1) {
            this.index = 0
          } else {
            ++this.index
          }
        },
        'random': () => {
          this.index = Math.floor(this.songList.length * Math.random())
        },
        'cycle': () => {
          if (this.index === this.songList.length - 1) {
            this.index = 0
          } else {
            ++this.index
          }
        }
      }
      t[this.playMode].call(this)
      this.currentSong = this.songList[this.index]
    },
    switchState () {
      this.playing = !this.playing
    },
    changeMode () {
      let nextMode = {
        'order': 'random',
        'random': 'cycle',
        'cycle': 'order'
      }
      this.playMode = nextMode[this.playMode]
    },
    renderLyrics () {
      this.$refs.lyrics.renderLyrics()
    },
    checkLyric (time) {
      this.$refs.lyrics.checkLyric(time)
    }
  },
  computed: {
    backgroundObj () {
      return {
        backgroundImage: `url(${this.currentSong.albumpic_big})`
      }
    }
  },
  created () {
    this.default()
  }
}
</script>

<style lang="scss">
$bodyBack: #42474C;
$maskBack: rgba(0, 0, 0, .35);
$scrollColor: rgba(255, 255, 255, .1);
$scrollWrapperColor: rgba(255, 255, 255, .1);
$searchColor: rgba(255, 255, 255, .07);
$searchBtnHoverColor: rgba(255, 255, 255, .1);
$textColor: rgba(225, 225, 225, .8);
$icon: 'http://7xrkxs.com1.z0.glb.clouddn.com/music/icon_sprite.png';
body,
html {
    height: 100%;
    overflow: hidden;
    background: $bodyBack;
    .iScrollVerticalScrollbar {
        position: absolute;
        z-index: 9999;
        width: 7px;
        bottom: 10px;
        top: 10px;
        right: 0px;
        pointer-events: none;
        background: $scrollWrapperColor;
        .iScrollIndicator {
            box-sizing: border-box;
            position: absolute;
            border-radius: 3px;
            width: 100%;
            transition-duration: 0ms;
            display: block;
            transform: translate(0px, 0px) translateZ(0px);
            transition-timing-function: cubic-bezier(0.1, 0.57, 0.1, 1);
            background: $scrollColor;
        }
    }
    .app {
        height: 100%;
        .backImg {
            position: absolute;
            width: 100%;
            height: 100%;
            background-repeat: no-repeat;
            background-position: 50%;
            background-size: cover;
            filter: blur(90px);
            opacity: 0.6
        }
        .mask {
            position: absolute;
            width: 100%;
            height: 100%;
            background: $maskBack;
        }
        .main-content {
            position: absolute;
            top: 80px;
            bottom: 165px;
            left: 0;
            right: 0;
            padding-top: 100px;
            width: 1100px;
            margin: auto;
            font-size: 0;
            .search-container {
                position: absolute;
                width: 100%;
                margin-top: -100px;
                text-align: center;
                height: 70px;
                .search-wrapper {
                    position: relative;
                    display: inline-block;
                    box-sizing: border-box;
                    height: 50px;
                    width: 700px;
                    padding-left: 10px;
                    padding-right: 50px;
                    background: $searchColor;
                    transition: 0.15s;
                    .search {
                        width: 100%;
                        height: 100%;
                        color: $textColor;
                        font-size: 16px;
                        background: transparent;
                        border: none;
                        outline: none;
                        &::-moz-placeholder {
                            color: $textColor;
                        }
                        &::-webkit-input-placeholder {
                            color: $textColor;
                        }
                        &:-ms-input-placeholder {
                            color: $textColor;
                        }
                    }
                    .search-btn {
                        position: absolute;
                        top: 0;
                        right: 0;
                        width: 50px;
                        height: 50px;
                        background: transparent;
                        border: none;
                        padding: 0;
                        cursor: pointer;
                        outline: none;
                        &:hover {
                            background: $searchBtnHoverColor;
                        }
                        .search-icon {
                            display: inline-block;
                            width: 16px;
                            height: 16px;
                            background: url($icon);
                            background-position: 0 -40px;
                        }
                    }
                }
            }
            .list-wrapper {
                position: relative;
                display: inline-block;
                padding-right: 25px;
                width: 700px;
                height: 100%;
                overflow: hidden;
            }
            .lyrics-wrapper {
                vertical-align: top;
                margin-left: 35px;
                width: 340px;
                display: inline-block;
                height: 100%;
            }
        }
        .player-wrapper {
            position: fixed;
            bottom: 20px;
            width: 100%;
            display: flex;
            justify-content: center;
        }
    }
}
</style>
