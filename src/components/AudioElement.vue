<template>
  <vue-plyr
    ref="plyr"
    :emit="['canplay', 'timeupdate', 'ended', 'seeked', 'playing', 'waiting', 'pause']"
    @canplay="onCanplay()"
    @timeupdate="onTimeupdate()"
    @ended="onEnded()"
    @seeked="onSeeked()"
    @playing="onPlaying()"
    @waiting="onWaiting()"
    @pause="onPause()"
  >
    <audio crossorigin="anonymous" >
      <source v-if="source" :src="source" />
    </audio>
  </vue-plyr>
</template>

<script>
import Lyric from 'lrc-file-parser'
import { mapState, mapGetters, mapMutations } from 'vuex'
import NotifyMixin from '../mixins/Notification.js'

export default {
  name: 'AudioElement',

  mixins: [NotifyMixin],

  data() {
    return {
      lrcObj: null,
      lrcAvailable: false,
    }
  },

  computed: {
    player () {
      return this.$refs.plyr.player
    },

    source () {
      // 从 LocalStorage 中读取 token
      const token = this.$q.localStorage.getItem('jwt-token') || ''
      // New API
      if (this.currentPlayingFile.mediaStreamUrl) {
        return `${this.currentPlayingFile.mediaStreamUrl}?token=${token}`
      } else if (this.currentPlayingFile.hash) {
        // Fallback to be compatible with old backend
        return `/api/media/stream/${this.currentPlayingFile.hash}?token=${token}`
      } else {
        return ""
      }
    },

    ...mapState('AudioPlayer', [
      'playing',
      'queue',
      'queueIndex',
      'playMode',
      'muted',
      'volume',
      // 'volumeAll',
      // 'volumeWork',
      'sleepTime',
      'sleepMode',
      'rewindSeekTime',
      'forwardSeekTime',
      'rewindSeekMode',
      'forwardSeekMode'
    ]),

    ...mapGetters('AudioPlayer', [
      'currentPlayingFile'
    ])
  },

  watch: {
    playing (flag) {
      if (this.player.duration) {
        // 缓冲至可播放状态
        flag ? this.player.play() : this.player.pause()
      }
      // this.playLrc(flag);
    },

    // watch source -> media.load() -> canPlay -> player.play()
    source (url) {
      if (url) {
        // 加载新音频/视频文件
        this.player.media.load();
        this.loadLrcFile();
      }
    },

    muted (flag) {
      // 切换静音状态
      this.player.muted = flag
    },
    // 真正影响音量的地方   这个应该是个后置相应
    // volume setting last third
    volume (val) {
      console.log('element  volume 真实设定  旧值 ' +this.player.volume+ '   新值：' +val)
      if (val < 0 || val > 10) {
        return
      }
      this.player.volume = val
    },
    // volumeAll (val) {
    //   console.log('element  volumeAll 真实设定  旧值 ' +this.player.volumeAll+ '   新值：' +val)
    //   if (val < 0 || val > 100) {
    //     return
    //   }
    //   this.player.volumeAll = val
    // },
    // volumeWork (val) {
    //   console.log('element 真实设定 volumeWork 旧值 ' +this.player.volumeWork+ '   新值：' +val)
    //   if (val < -9 || val > 9) {
    //     return
    //   }
    //   this.player.volumeWork = val
    // },

    rewindSeekMode(rewind) {
      if (rewind) {
        this.player.rewind(this.rewindSeekTime);
        this.SET_REWIND_SEEK_MODE(false);
      }
    },
    forwardSeekMode(forward) {
      if (forward) {
        this.player.forward(this.forwardSeekTime);
        this.SET_FORWARD_SEEK_MODE(false);
      }
    }
  },

  methods: {
    /**
     * 当 外部暂停（线控暂停、软件切换）、用户控制暂停、seek 时会触发本事件
     */
    onPause() {
      // console.log('onPause')
      this.playLrc(false)
      this.PAUSE()
    },
    /**
     * 当播放器真正开始播放时会触发本事件
     */
    onPlaying() {
      // console.log('playing')
      this.playLrc(true)
      this.PLAY()
    },
    /**
     * 当播放器缓冲区空，被迫暂停加载时会触发本事件
     */
    onWaiting() {
      // console.log('waiting')
      this.playLrc(false)
      this.PLAY()
    },
    ...mapMutations('AudioPlayer', [
      'SET_DURATION',
      'SET_CURRENT_TIME',
      'PAUSE',
      'PLAY',
      'SET_TRACK',
      'NEXT_TRACK',
      'SET_CURRENT_LYRIC',
      // 'SET_VOLUME', 无影响
      // 'SET_VOLUME_ALL',
      // 'SET_VOLUME_WORK',
      'CLEAR_SLEEP_MODE',
      'SET_REWIND_SEEK_MODE',
      'SET_FORWARD_SEEK_MODE'
    ]),

    onCanplay () {
      // 缓冲至可播放状态时触发 (只有缓冲至可播放状态, 才能获取媒体文件的播放时长)
      this.SET_DURATION(this.player.duration)

      // 播放
      if (this.playing && this.player.currentTime !== this.player.duration) {
        this.player.play()
      }
    },

    onTimeupdate () {
      // 当目前的播放位置已更改时触发
      this.SET_CURRENT_TIME(this.player.currentTime)
      if (this.sleepMode && this.sleepTime) {
        const currentTime = new Date()
        const currentHourStr = currentTime.getHours().toString().padStart(2, '0')
        const currentMinuteStr = currentTime.getMinutes().toString().padStart(2, '0')
        const sleepHourStr = this.sleepTime.match(/\d+/g)[0]
        const sleepMinuteStr = this.sleepTime.match(/\d+/g)[1]
        if (currentHourStr === sleepHourStr && currentMinuteStr === sleepMinuteStr) {
          this.PAUSE()
          this.CLEAR_SLEEP_MODE()
          // Persist sleep mode settings
          this.$q.sessionStorage.set('sleepTime', null)
          this.$q.sessionStorage.set('sleepMode', false)
        }
      }
    },

    onEnded () {
      // 当前文件播放结束时触发
      switch (this.playMode.name) {
        case "all repeat":
          // 循环播放
          if (this.queueIndex === this.queue.length - 1) {
            this.SET_TRACK(0)
          } else {
            this.NEXT_TRACK()
          }
          break
        case "repeat once":
          // 单曲循环
          this.player.currentTime = 0
          this.player.play()
          this.PLAY()
          break
        case "shuffle": {
          // 随机播放
          const index = Math.floor(Math.random()*this.queue.length)
          this.SET_TRACK(index)
          if (index === this.queueIndex) {
            this.player.currentTime = 0
          }
          break
        }
        default:
          // 顺序播放
          if (this.queueIndex === this.queue.length - 1) {
            this.PAUSE()
          } else {
            this.NEXT_TRACK()
          }
      }
    },

    onSeeked() {
      // if (this.lrcAvailable) {
      //   this.lrcObj.play(this.player.currentTime * 1000);
      //   if (!this.playing) {
      //     this.lrcObj.pause();
      //   }
      // }
    },


    playLrc (playStatus) {
      if (this.lrcAvailable) {
        if (playStatus) {
          this.lrcObj.play(this.player.currentTime * 1000);
        } else {
          this.lrcObj.pause();
        }
      }
    },

    initLrcObj () {
        this.lrcObj = new Lyric({
          onPlay: (line, text) => {
            this.SET_CURRENT_LYRIC(text);
          },
        })
    },

    loadLrcFile () {
      const token = this.$q.localStorage.getItem('jwt-token') || '';
      const fileHash = this.queue[this.queueIndex].hash;
      const url = `/api/media/check-lrc/${fileHash}?token=${token}`;

      this.$axios.get(url)
        .then((response) => {
          if (response.data.result) {
            // 有lrc歌词文件
            this.lrcAvailable = true;
            console.log('读入歌词');
            const lrcUrl = `/api/media/stream/${response.data.hash}?token=${token}`;
            this.$axios.get(lrcUrl)
              .then(response => {
                console.log('歌词读入成功');
                this.lrcObj.setLyric(response.data);
                this.lrcObj.play(this.player.currentTime * 1000);
              });
          } else {
            // 无歌词文件
            this.lrcAvailable = false;
            this.lrcObj.setLyric('');
            this.SET_CURRENT_LYRIC('');
          }
        })
        .catch((error) => {
          if (error.response) {
            // 请求已发出，但服务器响应的状态码不在 2xx 范围内
            if (error.response.status !== 401) {
              this.showErrNotif(error.response.data.error || `${error.response.status} ${error.response.statusText}`);
            }
          } else {
            this.showErrNotif(error.message || error);
          }
        })
    },
  },
  mounted () {
    console.log("element 的 mounted 方法调用")
    // 这俩是展示的， 还得找找实际的音量控制地方
    // this.AudioPlayer.SET_VOLUME(this.player.volume); // 报错
    this.initLrcObj();
    if (this.source) {
      this.loadLrcFile();
    }
  }
}
</script>
