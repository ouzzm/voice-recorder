<template>
  <v-slide-x-transition>
    <v-row v-show="cardSlider" justify="center" align="center">
      <v-col cols="12" sm="3" md="3" lg="3" xl="3">
        <v-card elevation="12">
          <v-card-title style="background: #8a2be2">
            <v-slide-y-transition>
              <v-img
                v-show="imageSlider"
                lazy-src="/logo.png"
                src="/logo.png"
                max-width="140"
                max-height="140"
                contain
              />
            </v-slide-y-transition>
          </v-card-title>
          <v-card-text class="pa-0 py-4 text-center">
            <v-slide-y-reverse-transition>
              <v-row
                v-show="recordSlider"
                class="ma-0"
                justify="center"
                align="center"
              >
                <v-col class="pa-0" cols="12" sm="12" md="12" lg="12" xl="12">
                  <v-progress-circular
                    :indeterminate="recording"
                    size="125"
                    width="10"
                    color="#f16a43"
                  >
                    <v-icon color="#8a2be2" size="65" @click="Record">
                      {{ !recording ? 'mdi-microphone' : 'mdi-stop' }}
                    </v-icon>
                  </v-progress-circular>
                  <br />
                  <h2 v-if="recording" class="mt-2">
                    {{ recordTime }} Recording...
                  </h2>
                </v-col>
              </v-row>
            </v-slide-y-reverse-transition>
            <v-slide-y-transition>
              <v-row
                v-show="listenSlider"
                class="ma-0"
                justify="center"
                align="center"
              >
                <v-col class="pa-0" cols="4" sm="4" md="4" lg="4" xl="4">
                  <v-progress-circular size="125" width="10" color="#f16a43">
                    <v-icon color="#8a2be2" size="55" @click="$router.go(0)">
                      {{ 'mdi-replay' }}
                    </v-icon>
                  </v-progress-circular>
                </v-col>
                <v-col class="pa-0" cols="4" sm="4" md="4" lg="4" xl="4">
                  <v-progress-circular
                    rotate="360"
                    size="125"
                    width="10"
                    :value="listenLoading"
                    color="#f16a43"
                  >
                    <v-icon color="#8a2be2" size="65" @click="Listen">
                      {{ !listening ? 'mdi-play' : 'mdi-stop' }}
                    </v-icon>
                  </v-progress-circular>
                </v-col>
                <v-col class="pa-0" cols="4" sm="4" md="4" lg="4" xl="4">
                  <h1 class="mb-4">{{ listenTime }}</h1>
                  <h1 class="mt-4">{{ recordTime }}</h1>
                </v-col>
                <v-col class="pb-0" cols="12" sm="12" md="12" lg="12" xl="12">
                  <!-- <v-btn
                    class="text-capitalize"
                    color="#8a2be2"
                    width="90"
                    rounded
                    dark
                  >
                    QR Code
                  </v-btn> -->
                  <v-btn
                    class="text-capitalize mr-2"
                    color="#f16a43"
                    width="170"
                    rounded
                    large
                    dark
                    @click="Download"
                  >
                    <v-icon left>mdi-download</v-icon> Download
                  </v-btn>
                  <!-- <v-btn
                    class="text-capitalize"
                    color="#8a2be2"
                    width="90"
                    rounded
                    dark
                  >
                    Share
                  </v-btn> -->
                  <v-btn
                    class="text-capitalize ml-2"
                    color="#f16a43"
                    width="170"
                    rounded
                    large
                    dark
                    @click="$router.go(0)"
                  >
                    <v-icon left>mdi-delete</v-icon>Delete
                  </v-btn>
                </v-col>
              </v-row>
            </v-slide-y-transition>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-slide-x-transition>
</template>

<script>
import WaveformData from 'waveform-data'
import Mp3Encoder from '~/utils/mp3Encoder'
export default {
  data() {
    return {
      cardSlider: false,
      imageSlider: false,
      recordSlider: true,
      recording: false,
      recordTime: '00:00',
      recorder: null,
      stream: null,
      audioContext: false,
      file: false,
      error: false,
      errorText: '',
      soundFile: null,
      activeSound: null,
      activePlay: false,
      activePlayId: 0,
      listenSlider: false,
      listening: false,
      listenTime: '00:00',
      listenLoading: 0,
      interval: null,
      supportedTypes: ['audio/wav', 'audio/ogg', 'audio/mp3', 'audio/mpeg'],
    }
  },
  head() {
    return {
      title: 'Record Your Voice',
    }
  },
  watch: {
    error(newval) {
      if (!newval) {
        this.$refs.file.value = null
      }
    },
  },
  created() {
    setTimeout(() => {
      this.cardSlider = true
    }, 400)
    setTimeout(() => {
      this.imageSlider = true
    }, 600)
    setTimeout(() => {
      this.recordSlider = true
    }, 800)
  },
  mounted() {
    window.onload = () => {
      this.audioContext = new (window.AudioContext ||
        window.webkitAudioContext)()
    }
  },
  methods: {
    Record() {
      const comp = this

      this.audioContext = new (window.AudioContext ||
        window.webkitAudioContext)()

      this.processor = this.audioContext.createScriptProcessor(4096, 1, 1)

      navigator.getUserMedia =
        navigator.getUserMedia ||
        navigator.webkitGetUserMedia ||
        navigator.mozGetUserMedia ||
        navigator.msGetUserMedia

      if (this.recording) {
        this.recording = false
        clearInterval(this.interval)

        this.recordSlider = !this.recordSlider
        setTimeout(() => {
          this.listenSlider = !this.listenSlider
        }, 200)

        this.stream.getAudioTracks().forEach((c) => {
          c.stop()
        })
      } else {
        navigator.getUserMedia(
          {
            audio: true,
          },
          (stream) => {
            comp.stream = stream
            comp.input = comp.audioContext.createMediaStreamSource(stream)
            comp.input.connect(comp.processor)
            comp.processor.connect(comp.audioContext.destination)

            comp.recorder = new MediaRecorder(stream, {
              audioBitsPerSecond: 128000,
              bitsPerSecond: 128000,
              mimeType: 'audio/webm;codecs=opus',
            })

            if (!comp.recording) {
              comp.error = false
              comp.errorText = null

              comp.recorder.start()
              comp.recording = true

              const countDownDate = new Date()
              comp.interval = setInterval(() => {
                const now = new Date().getTime()

                const distance = now - countDownDate.getTime()

                let minutes = Math.floor(
                  (distance % (1000 * 60 * 60)) / (1000 * 60)
                )
                let seconds = Math.floor((distance % (1000 * 60)) / 1000)

                minutes = String(minutes).length >= 2 ? minutes : '0' + minutes
                seconds = String(seconds).length >= 2 ? seconds : '0' + seconds

                comp.recordTime = minutes + ':' + seconds
              }, 1000)

              const chunks = []

              const mp3Encoder = new Mp3Encoder({
                bitRate: 700,
                sampleRate: '44100',
              })

              comp.processor.onaudioprocess = (ev) => {
                const sample = ev.inputBuffer.getChannelData(0)

                mp3Encoder.encode(sample)
              }

              comp.recorder.ondataavailable = (data) => {
                chunks.push(data.data)
              }

              comp.recorder.onstop = () => {
                comp.stream.getAudioTracks().forEach((c) => {
                  c.stop()
                })

                comp.input.disconnect()
                comp.processor.disconnect()

                // eslint-disable-next-line no-unused-vars
                const blob = new Blob(chunks, {
                  type: 'audio/webm',
                })

                const c = mp3Encoder.finish()

                comp.ReadFileAndAdd(c.blob, true, comp.recordTime)
              }
            }
          },
          (error) => {
            comp.error = true
            comp.errorText =
              'Need Permission.Please confirm audio permission from browser prompt'
            console.log(error)
          }
        )
      }
    },
    ReadFileAndAdd(file, record, duration) {
      if (!this.file) {
        this.file = new FileReader()
      }

      this.file.readAsDataURL(file)

      const musicFile = {
        title: record ? 'xxx' : file.name,
      }

      const comp = this

      this.file.onload = (event) => {
        musicFile.base64 = comp.file.result.toString()

        const audio = document.createElement('audio')
        audio.src = event.target.result

        const snd = new Audio(musicFile.base64)

        snd.addEventListener('loadeddata', () => {
          const createBuffer = require('audio-buffer-from')
          const crypto = require('crypto')
          const buffer = createBuffer(musicFile.base64, 'uint8')

          const options = {
            audio_context: this.audioContext,
            audio_buffer: buffer,
            scale: 186,
          }

          WaveformData.createFromAudio(options, (error, waveformx) => {
            const channel = waveformx.channel(0)

            const waveform = {
              version: 2,
              channels: waveformx.channels,
              sample_rate: waveformx.sample_rate,
              samples_per_pixel: waveformx.scale,
              bits: waveformx.bits,
              length: waveformx.length,

              data: [],
            }

            for (let i = 0; i < waveformx.length; i++) {
              for (let channel = 0; channel < waveformx.channels; channel++) {
                waveform.data.push(waveformx.channel(channel).min_sample(i))
                waveform.data.push(waveformx.channel(channel).max_sample(i))
              }
            }

            musicFile.waveForm = {
              waveform,
              max: channel.max_array(),
              min: channel.min_array(),
              channel,
              length: waveformx.length,
            }

            musicFile.duration = duration
            musicFile.id = crypto.randomBytes(16).toString('hex')

            comp.soundFile = musicFile

            console.log(error)
          })
        })
      }
    },
    Listen() {
      this.listening = !this.listening

      if (this.listening) {
        // play
        this.listenTime = '00:00'

        this.activeSound = new Audio(this.soundFile.base64)

        this.activePlay = true
        this.activePlayId = this.soundFile.id

        this.activeSound.play()
        this.activeSound.onended = () => {
          this.activePlayId = false
          this.activePlay = false
        }

        const countDownDate = new Date()

        this.interval = setInterval(() => {
          if (this.listenTime !== this.recordTime) {
            const now = new Date().getTime()

            const distance = now - countDownDate.getTime()

            let minutes = Math.floor(
              (distance % (1000 * 60 * 60)) / (1000 * 60)
            )
            let seconds = Math.floor((distance % (1000 * 60)) / 1000)

            minutes = String(minutes).length >= 2 ? minutes : '0' + minutes
            seconds = String(seconds).length >= 2 ? seconds : '0' + seconds

            this.listenTime = minutes + ':' + seconds

            const a =
              this.listenTime.split(':')[0] * 60 + this.listenTime.split(':')[1]
            const b =
              this.recordTime.split(':')[0] * 60 + this.recordTime.split(':')[1]

            this.listenLoading = (100 * a) / b
          } else {
            clearInterval(this.interval)
            this.listening = !this.listening
          }
        }, 1000)
      } else {
        // pause
        this.activeSound.pause()
        this.activeSound.currentTime = 0.0

        this.activePlayId = null
        this.activePlay = false

        clearInterval(this.interval)
        this.listenTime = '00:00'
        this.listenLoading = 0
      }
    },
    Download() {
      const a = document.createElement('a')
      a.href = this.soundFile.base64
      a.download = 'record'
      a.click()
    },
  },
}
</script>
