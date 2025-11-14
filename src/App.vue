<template>
  <div>
    <!-- 本地视频 -->
    <div
      ref="localVideoElement"
      class="video-container"
      :class="{'video-container--muted': localVideoTrack?.isMuted}"
    ></div>
    <div class="controls">
      <button @click="toggleVideo">
        {{ localVideoTrack?.isMuted ? '开启视频' : '关闭视频' }}
      </button>
      <button @click="toggleAudio" :disabled="!localAudioTrack">
        {{ localAudioTrack?.isMuted ? '开启音频' : '关闭音频' }}
      </button>
      <button @click="connectRoom" :disabled="isConnected">连接房间</button>
      <button @click="disconnectRoom" :disabled="!isConnected">断开连接</button>
    </div>

    <!-- 远程视频 -->
    <div class="video-section">
      <h3>远程参与者</h3>
      <div v-for="participant in remoteParticipants" :key="participant.identity" class="participant">
        <div
          :ref="`participant-video-${participant.identity}`"
          class="video-container"
          :class="{
            'video-container--active': participant.tracks.video,
            'video-container--muted': participant.tracks.videoMuted
          }"
        ></div>
        <div
          :ref="`participant-audio-${participant.identity}`"
          class="audio-container"
          :data-active="participant.tracks.audio"
        ></div>
      </div>
    </div>
  </div>
</template>

<script>
import {Room, createLocalAudioTrack, createLocalVideoTrack } from 'livekit-client'

export default {
  name: 'App',
  data() {
    return {
      url: "",
      token: "",
      room: null,
      isConnected: false,
      localVideoTrack: null,
      localAudioTrack: null,
      remoteParticipants: []
    }
  },
  async mounted() {
    let urlOptions = this.getQueryParams()
    if (urlOptions.roomServer) this.url = urlOptions.roomServer
    if (urlOptions.roomToken) this.token = urlOptions.roomToken

    this.room = new Room()
    this.setupRoomListeners()
  },
  beforeUnmount() {
    this.disconnectRoom()
  },
  methods: {
    getQueryParams() {
      const queryString = window.location.search
      const urlParams = new URLSearchParams(queryString)
      const params = {}
      for (const [key, value] of urlParams.entries()) {
        params[key] = value
      }
      return params
    },

    setupRoomListeners() {
      const room = this.room

      room.on('participantConnected', participant => {
        console.log("远程参与者加入:", participant.identity)
        this.ensureRemoteParticipantEntry(participant)
      })

      room.on('participantDisconnected', participant => {
        console.log("远程参与者离开:", participant.identity)
        const index = this.remoteParticipants.findIndex(p => p.identity === participant.identity)
        if (index > -1) this.remoteParticipants.splice(index, 1)
      })

      room.on('trackSubscribed', (track, publication, participant) => {
        console.log('远程轨道订阅:', track.kind, participant?.identity)
        const entry = this.ensureRemoteParticipantEntry(participant)
        if (!entry) return
        entry.tracks[track.kind] = true
        const mutedKey = track.kind === 'video' ? 'videoMuted' : 'audioMuted'
        entry.tracks[mutedKey] = track.isMuted ?? publication.isMuted ?? false
        this.attachRemoteTrack(track, participant)
      })

      room.on('trackMuted', (pub, participant) => {
        console.log('远程轨道静音:', pub.kind, participant?.identity)
        this.handleRemoteTrackMuteChange(pub, participant, true)
      })

      room.on('trackUnmuted', (pub, participant) => {
        console.log('远程轨道取消静音:', pub.kind, participant?.identity)
        this.handleRemoteTrackMuteChange(pub, participant, false)
      })

      room.on('trackUnsubscribed', (track, publication, participant) => {
        console.log('远程轨道取消订阅:', track.kind, participant?.identity)
        const entry = this.ensureRemoteParticipantEntry(participant)
        if (!entry) return
        entry.tracks[track.kind] = false
        if (track.kind === 'video') {
          entry.tracks.videoMuted = false
        } else {
          entry.tracks.audioMuted = false
        }
        this.detachRemoteTrack(track, participant)
      })
    },

    ensureRemoteParticipantEntry(participant) {
      if (!participant || participant.isLocal) return null
      let entry = this.remoteParticipants.find(p => p.identity === participant.identity)
      if (!entry) {
        entry = {
          identity: participant.identity,
          tracks: {
            video: false,
            audio: false,
            videoMuted: false,
            audioMuted: false
          }
        }
        this.remoteParticipants.push(entry)
      }
      return entry
    },

    handleRemoteTrackMuteChange(pub, participant, isMuted) {
      const entry = this.ensureRemoteParticipantEntry(participant)
      if (!entry) return
      const refKey = pub.kind === 'video'
        ? `participant-video-${entry.identity}`
        : `participant-audio-${entry.identity}`
      if (pub.kind === 'video') {
        entry.tracks.videoMuted = isMuted
      } else {
        entry.tracks.audioMuted = isMuted
      }

      this.$nextTick(() => {
        const elList = this.$refs[refKey]
        if (!elList || !elList[0]) return
        const el = elList[0]
        const media = el.querySelector(pub.kind === 'video' ? 'video' : 'audio')
        if (media) media.style.opacity = isMuted ? '0.5' : '1'
      })
    },

    async ensureMediaDevicesAccess() {
      if (!navigator.mediaDevices || typeof navigator.mediaDevices.getUserMedia !== 'function') {
        throw new Error('浏览器不支持媒体设备')
      }
      try {
        const testStream = await navigator.mediaDevices.getUserMedia({video: true, audio: true})
        testStream.getTracks().forEach(t => t.stop())
      } catch (e) {
        console.error('获取媒体设备失败:', e)
      }
    },

    async connectRoom() {
      try {
        await this.room.connect(this.url, this.token)
        this.isConnected = true
        await this.ensureMediaDevicesAccess()

        this.localVideoTrack = await createLocalVideoTrack()
        this.localAudioTrack = await createLocalAudioTrack()

        await this.room.localParticipant.publishTrack(this.localVideoTrack, {name: 'camera'})
        await this.room.localParticipant.publishTrack(this.localAudioTrack, {name: 'microphone'})

        this.renderLocalVideo()
      } catch (error) {
        console.error('连接房间失败:', error)
      }
    },

    async disconnectRoom() {
      if (this.room) await this.room.disconnect()
      this.isConnected = false
      this.localVideoTrack = null
      this.localAudioTrack = null
      this.remoteParticipants = []
    },

    // 切换视频
    async toggleVideo() {
      const pub = this.room.localParticipant.getTrackPublication('camera')
      if (!pub || !pub.track) return
      const track = pub.track

      if (track.isMuted) {
        await track.unmute()
        this.renderLocalVideo()
      } else {
        await track.mute()
      }

      this.$forceUpdate()
    },

    // 切换音频
    async toggleAudio() {
      const pub = this.room.localParticipant.getTrackPublication('microphone')
      if (!pub || !pub.track) return
      const track = pub.track

      if (track.isMuted) {
        await track.unmute()
      } else {
        await track.mute()
      }

      this.$forceUpdate()
    },

    attachTrack(track, element) {
      if (!track || !element) return
      const selector = track.kind === 'video' ? 'video' : 'audio'
      this.clearMediaElement(element, selector)

      const mediaEl = document.createElement(selector)
      mediaEl.autoplay = true
      mediaEl.playsInline = true
      if (track.kind === 'video') {
        mediaEl.muted = true
        mediaEl.style.opacity = '1'
      }

      mediaEl.srcObject = new MediaStream([track.mediaStreamTrack])
      element.appendChild(mediaEl)
    },

    attachRemoteTrack(track, participant) {
      const refKey = track.kind === 'video'
        ? `participant-video-${participant.identity}`
        : `participant-audio-${participant.identity}`

      this.$nextTick(() => {
        const elList = this.$refs[refKey]
        if (elList && elList[0]) this.attachTrack(track, elList[0])
      })
    },

    detachRemoteTrack(track, participant) {
      const refKey = track.kind === 'video'
        ? `participant-video-${participant.identity}`
        : `participant-audio-${participant.identity}`

      this.$nextTick(() => {
        const elList = this.$refs[refKey]
        if (!elList || !elList[0]) return
        const el = elList[0]
        const selector = track.kind === 'video' ? 'video' : 'audio'
        this.clearMediaElement(el, selector)
      })
    },

    clearMediaElement(element, selector) {
      if (!element) return
      const mediaEls = element.querySelectorAll(selector)
      mediaEls.forEach(node => {
        node.srcObject = null
        node.remove()
      })
    },

    renderLocalVideo() {
      this.$nextTick(() => {
        const container = this.$refs.localVideoElement
        if (this.localVideoTrack && container) {
          this.attachTrack(this.localVideoTrack, container)
        }
      })
    }
  }
}
</script>

<style>
html, body {
  margin: 0;
  width: 100%;
  height: 100%;
}

.video-section {
  margin-bottom: 30px;
}

.video-container {
  width: 100%;
  height: 300px;
  background-color: #000;
  margin-bottom: 10px;
}

.video-container video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.video-container--muted {
  background-color: #000 !important;
}

.video-container--muted video {
  opacity: 0 !important;
}

.controls {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

button {
  padding: 8px 16px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:disabled {
  background-color: #6c757d;
  cursor: not-allowed;
}

.participant {
  width: 100%;
  display: inline-block;
  margin-right: 20px;
}

.audio-container {
  width: 0;
  height: 0;
  overflow: hidden;
}

.video-container--active {
  background-color: transparent;
}
</style>
