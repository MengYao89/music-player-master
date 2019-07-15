<template>
  <div class="player-container">
    <audio ref="audio" :src="song.m4a"></audio>
    <div class="left-content">
      <span @click="prevSong()" class="prev bg-icon"></span>
      <span class="bg-icon" :class="controlClass" @click="switchState"></span>
      <span @click="nextSong()" class="next bg-icon"></span>
    </div>
    <div class="right-content">
      <div class="intro">
        <span class="title">{{ song.songname }}</span>
        <span class="singer">{{ song.singername }}</span>
      </div>
      <div class="progress-wrapper">
        <div class="time">
          <span class="currTime">{{ getTime(currLen) }}</span>
          <span class="totalTime">/ {{ getTime(totalLen) }}</span>
        </div>
        <div class="volume-wrapper">
          <span class="volume-btn" :class="volumeClass" @click="changeMute"></span>
          <div @click="changeVolPos" class="volume-bar">
            <div :style="{width: volPos.btn + 'px'}" class="current-bar"></div>
            <div @click.stop="stopPass" @mousedown.stop.prevent="drag($event, 'volPos')" :style="{left: volPos.btn + 'px'}" class="controller"></div>
          </div>
        </div>
        <span class="play_mod-icon" :class="play_modClass" @click="changeMode"></span>
        <div class="progress_bar-wrapper">
          <div @click="changeProgress" ref="progressBar" class="progress_bar">
            <div :style="{width: progressPos.btn + 'px'}" class="current-progress"></div>
          </div>
          <div @mousedown.stop.prevent="drag($event, 'progressPos')" :style="{left: progressPos.btn + 'px'}" class="progress-btn"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  export default {
    props: {
      song: {
        type: Object
      },
      playing: {
        type: Boolean
      },
      playMode: {
        type: String
      }
    },
    watch: {
      song () {
        clearInterval(this.intervalId)
      },
      playing () {
        if (this.playing) {
          this.$refs.audio.play()
        } else {
          this.$refs.audio.pause()
        }
      }
    },
    data () {
      return {
        volPos: {
          max: 80,
          btn: 80
        },
        progressPos: {
          max: 0,
          btn: 0
        },
        posData: {
          mouseStartX: '',
          btnStartX: ''
        },
        btnType: '',
        totalLen: 0,
        currLen: 0,
        intervalId: '',
        latestVolume: 80
      }
    },
    methods: {
      switchState () {
        this.$emit('switchState')
      },
      drag (e, type) {
        this.posData.mouseStartX = e.pageX
        this.posData.btnStartX = this[type].btn
        this.btnType = type
        this.btnDownManager()
        window.addEventListener('mousemove', this._mousemoveCallback)
        window.addEventListener('mouseup', this._mouseupCallback)
      },
      _mousemoveCallback (e) {
        (this._throttleV2(() => {
          this.moveBtn(e)
          this.btnMoveManager()
        }, 15, 30))()
      },
      _mouseupCallback () {
        window.removeEventListener('mousemove', this._mousemoveCallback)
        window.removeEventListener('mouseup', this._mouseupCallback)
        this.btnUpManager()
      },
      moveBtn (e) {
        let distanceX = e.pageX - this.posData.mouseStartX
        this[this.btnType].btn = this.posData.btnStartX + distanceX
        if (this[this.btnType].btn < 0) {
          this[this.btnType].btn = 0
        }
        if (this[this.btnType].btn > this[this.btnType].max) {
          this[this.btnType].btn = this[this.btnType].max
        }
      },
      _throttleV2 (fn, delay, mustRunDelay) {
        let timer = null
        let tStart
        return function () {
          let context = this
          let args = arguments
          let tCurr = +new Date()
          clearTimeout(timer)
          if (!tStart) {
            tStart = tCurr
          }
          if (tCurr - tStart >= mustRunDelay) {
            fn.apply(context, args)
            tStart = tCurr
          } else {
            timer = setTimeout(function () {
              fn.apply(context, args)
            }, delay)
          }
        }
      },
      btnDownManager () {
        let t = {
          'volPos': () => {
            return
          },
          'progressPos': () => {
            clearInterval(this.intervalId)
          }
        }
        t[this.btnType]()
      },
      btnMoveManager () {
        let t = {
          'volPos': () => {
            this.setVolume()
          },
          'progressPos': () => {
            this.currLen = ((this.progressPos.btn / this.progressPos.max) * this.totalLen).toFixed(4)
          }
        }
        t[this.btnType]()
      },
      btnUpManager () {
        let t = {
          'volPos': () => {
            return
          },
          'progressPos': () => {
            this.$refs.audio.currentTime = ((this.progressPos.btn / this.progressPos.max) * this.$refs.audio.duration).toFixed(4)
            this.setProgress()
          }
        }
        t[this.btnType]()
      },
      setVolume () {
        this.$refs.audio.volume = (this.volPos.btn / this.volPos.max).toFixed(2)
      },
      setProgress () {
        this.intervalId = setInterval(() => {
          if (this.$refs.audio.ended) {
            this.currLen = 0
            this.autoNextSong()
            return
          }
          this.currLen = this.$refs.audio.currentTime
          this.progressPos.btn = parseFloat(((this.currLen / this.totalLen) * this.progressPos.max).toFixed(2))
          if (this.playing) {
            this.$emit('checkLyric', this.currLen)
          }
        }, 100)
      },
      changeMute () {
        if (this.volPos.btn) {
          this.latestVolume = this.volPos.btn
          this.volPos.btn = 0
          this.setVolume()
        } else {
          this.volPos.btn = this.latestVolume
          this.setVolume()
        }
      },
      changeVolPos (e) {
        this.volPos.btn = e.offsetX
        this.setVolume()
      },
      stopPass () {
        return
      },
      initMusic () {
        this.$refs.audio.addEventListener('loadedmetadata', () => {
          this.$emit('renderLyrics')
          this.totalLen = this.$refs.audio.duration
          if (this.playing) {
            this.$refs.audio.play()
          }
          this.setProgress()
        })
      },
      getTime (seconds) {
        let m = Math.floor(seconds / 60)
        if (m < 10) {
          m = '0' + m
        }
        let s = Math.floor(seconds % 60)
        if (s < 10) {
          s = '0' + s
        }
        return m + ':' + s
      },
      changeProgress (e) {
        let length = e.offsetX
        this.$refs.audio.currentTime = ((length / this.progressPos.max) * this.totalLen).toFixed(2)
        this.currLen = this.$refs.audio.currentTime
      },
      prevSong () {
        this.$emit('prevSong')
      },
      autoNextSong () {
        let t = {
          order () {
            this.nextSong()
          },
          random () {
            this.nextSong()
          },
          cycle () {
            this.$refs.audio.currentTime = 0
            this.$refs.audio.play()
          }
        }
        t[this.playMode].call(this)
      },
      nextSong () {
        this.$emit('nextSong')
      },
      changeMode () {
        this.$emit('changeMode')
      }
    },
    computed: {
      controlClass () {
        const className = {
          play: 'play',
          pause: 'pause'
        }
        if (this.playing) {
          return className['pause']
        }
        return className['play']
      },
      volumeClass () {
        const className = {
          volumeOff: 'volume-off',
          volumeOn: 'volume-on'
        }
        if (this.volPos.btn === 0) {
          return className['volumeOff']
        }
        return className['volumeOn']
      },
      play_modClass () {
        const className = {
          order: 'order',
          random: 'random',
          cycle: 'cycle'
        }
        return className[this.playMode]
      }
    },
    mounted () {
      this.initMusic()
      this.progressPos.max = this.$refs.progressBar.clientWidth
    }
  }
</script>

<style lang="scss">
$textColor: rgba(225, 225, 225, .8);
$progressColor: rgba(225, 225, 225, .2);
$currentProgressColor: rgba(225, 225, 225, .7);
$volumeColor: rgba(225, 225, 225, .1);
$currentVolumeColor: rgba(225, 225, 225, .7);
$btnColor: rgba(225, 225, 225, .9);
$icon: 'http://7xrkxs.com1.z0.glb.clouddn.com/music/player.png';
.player-container {
    width: 1100px;
    height: 115px;
    font-size: 0;
    .left-content {
        display: inline-block;
        text-align: center;
        vertical-align: top;
        width: 200px;
        height: 100%;
        line-height: 115px;
        font-size: 0;
        .bg-icon {
            vertical-align: middle;
            display: inline-block;
            opacity: 0.8;
            cursor: pointer;
            background-image: url($icon);
        }
        .bg-icon:hover {
            opacity: 1
        }
        .bg-icon.prev {
            width: 19px;
            height: 20px;
            margin-right: 65px;
            background-position: 0 -30px;
        }
        .bg-icon.play {
            width: 21px;
            height: 29px;
            margin-right: 60px;
            background-position: 0 0;
        }
        .bg-icon.pause {
            width: 21px;
            height: 29px;
            margin-right: 60px;
            background-position: -30px 0;
        }
        .bg-icon.next {
            width: 19px;
            height: 20px;
            background-position: 0 -52px;
        }
    }
    .right-content {
        display: inline-block;
        vertical-align: top;
        width: 850px;
        height: 100%;
        margin-left: 50px;
        .intro {
            margin-top: 20px;
            font-size: 20px;
            .title {
                color: $textColor;
            }
            .singer {
                margin-left: 10px;
                color: $textColor;
                font-size: 20px;
            }
        }
        .progress-wrapper {
            position: relative;
            height: 25px;
            line-height: 25px;
            margin-top: 13px;
            color: $textColor;
            .time {
                font-size: 16px;
                display: inline-block;
                vertical-align: top;
            }
            .volume-wrapper {
                width: 130px;
                display: inline-block;
                margin-left: 16px;
                .volume-btn {
                    display: inline-block;
                    width: 26px;
                    height: 21px;
                    margin-top: 1px;
                    background: url($icon);
                    opacity: 0.8;
                }
                .volume-on {
                    background-position: 0 -144px;
                }
                .volume-off {
                    background-position: 0 -182px;
                }
                .volume-btn:hover {
                    opacity: 1;
                }
                .volume-bar {
                    position: absolute;
                    display: inline-block;
                    margin-top: 10px;
                    margin-left: 10px;
                    width: 80px;
                    height: 2px;
                    background: $volumeColor;
                    cursor: pointer;
                    .current-bar {
                        position: absolute;
                        left: 0;
                        top: 0;
                        height: 2px;
                        background: $currentVolumeColor;
                    }
                    .controller {
                        position: absolute;
                        top: 0;
                        margin-left: -5px;
                        margin-top: -4px;
                        width: 10px;
                        height: 10px;
                        background: $btnColor;
                        border-radius: 50%;
                        cursor: pointer;
                    }
                }
            }
            .play_mod-icon {
                position: absolute;
                display: inline-block;
                right: 0;
                top: 0;
                background: url($icon);
                opacity: 0.8;
                cursor: pointer;
                &.order {
                    width: 26px;
                    height: 25px;
                    background-position: 0 -205px;
                }
                &.random {
                    width: 25px;
                    height: 19px;
                    background-position: 0 -74px;
                }
                &.cycle {
                    width: 26px;
                    height: 25px;
                    background-position: 0 -232px;
                }
            }
            .play_mod-icon:hover {
                opacity: 1;
            }
            .progress_bar-wrapper {
                position: absolute;
                bottom: -15px;
                width: 100%;
                .progress_bar {
                    position: relative;
                    height: 2px;
                    background: $progressColor;
                    cursor: pointer;
                    overflow: hidden;
                    .current-progress {
                        position: absolute;
                        height: 2px;
                        background: $currentProgressColor;
                    }
                }
                .progress-btn {
                    position: absolute;
                    width: 10px;
                    height: 10px;
                    background: $btnColor;
                    border-radius: 50%;
                    margin-top: -6px;
                    margin-left: -5px;
                    cursor: pointer;
                }
            }
        }
    }
}
</style>
