<template>
  <div class="container mx-auto mt-2">
    <span>webrtc</span>
    <div class="w-1/2 flex flex-col">
      <video :srcObject="main.srcObject" autoplay controls muted></video>
      <span>main</span>
      <div class="w-full flex gap-4">
        <button @click="handleMic" class="w-1/2 py-2 px-3 rounded-md text-white h-auto transition-color" :class="main.enabled ? 'bg-green-700' : 'bg-red-700'">Mic</button>
        <button @click="handleShare" class="w-1/2 py-2 px-3 rounded-md text-white h-auto transition-color bg-blue-700">Share</button>
      </div>
    </div>
    <div class="w-1/2 flex">
      <div class="flex flex-col" v-for="(item, index) in child" :key="index" v-show="item.peerIdentity !== main.peerIdentity">
        <span>{{item.peerIdentity}}</span>
        <video v-if="item.status === 'connected'" :srcObject="item.srcObject" controls autoplay></video>
        <div class="py-2 px-3 rounded-md" v-else>{{item.status}}</div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

interface ERtcData {
  srcObject: MediaStream | null,
  phid: string;
  peerIdentity: string,
  pc: RTCPeerConnection | null,
  status: string,
  enabled?: boolean
}

export default defineComponent({
  data() {
    return {
      streamConfig: {
        video: false,
        audio: {
          echoCancellation: true,
          noiseSuppression: true
        }
      } as MediaStreamConstraints,
      main: {
        srcObject: null,
        phid: "PHID-USER-d3hkbnloohpnvwf2eswe",
        peerIdentity: "",
        pc: null,
        status: "",
        enabled: false
      } as ERtcData,
      child: [] as ERtcData[],
      ws: null as WebSocket | null,
      wsConfig: {
        organization: "suite",
        conpherence: "PHID-CONP-3g44bsbe2l6xznv54mvn"
      },
      rtcConfig: {
        iceServers: [
          {
            urls: "stun:ice.suiteapp.id:3478"
          }
        ]
      } as RTCConfiguration,
      share: undefined as RTCRtpSender | undefined
    }
  },
  async mounted() {
    await this.init()
  },
  methods: {
    async init() {
      // const videoStream = this.createDummyVideo()
      const audioStream = await navigator.mediaDevices.getUserMedia(this.streamConfig)
      // this.main.srcObject = new MediaStream([videoStream, audioStream.getAudioTracks()[0]])
      this.main.srcObject = audioStream
      this.main.srcObject.getAudioTracks()[0].enabled = false;
      this.connectSocket()
    },
    createDummyVideo() {
      const canvas = document.createElement("canvas") as any;
      canvas.width = 640;
      canvas.height = 480;
      canvas.getContext("2d").fillRect(0, 0, 640, 480);
      const stream = canvas.captureStream(30);
      return Object.assign(stream.getVideoTracks()[0], { enabled: true });
    },
    sendMessage(event: string, message: any, toPeer?: string) {
      const payload = {
        event,
        data: message,
        to: toPeer
      };
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify(payload));
      }
    },
    connectSocket() {
      this.ws = new WebSocket(
          `wss://ice.suiteapp.id:8443/websocket/${this.wsConfig.organization}-${this.wsConfig.conpherence}`
      );
      this.ws.onopen = () => {
        // first join page
        this.sendMessage("authenticate", {
          password: "tuuuuuurn",
          username: this.main.phid
        });

        if (this.ws) {
          this.ws.onmessage = event => {
            const { data } = event;
            const message = JSON.parse(data);
            if (
                message.data["peer_id"] &&
                message["event"] === "authenticated"
            ) {
              this.main.peerIdentity = message.data["peer_id"];
              this.initRTC(this.main.peerIdentity, this.main.phid, true);
            }
            if (message["event"] === "offer") {
              this.handleOffer(message);
            }
            if (message["event"] === "answer") {
              this.handleAnswer(message);
            }
            if (message["event"] === "candidate") {
              this.handleCandidate(message);
            }
            if (message["event"] === "joined") {
              this.handleJoined(message);
            }
            if (message["event"] === "left") {
              this.handleLeave(message);
            }
          };
        }
      };
      this.ws.onclose = () => this.connectSocket();
    },
    findIdx(peerIdentity: string) {
      return this.child.findIndex(data => data.peerIdentity === peerIdentity);
    },
    async initRTC(peerIdentity: string, phid: string, isInitiator: boolean) {
      try {
        const pc = new RTCPeerConnection(this.rtcConfig);
        pc.ontrack = this.handleRemoteStream(peerIdentity);
        pc.onicecandidate = this.handleIceCandidate(peerIdentity);
        // pc.oniceconnectionstatechange = (event: Event) => {
        //   console.log(event, peerIdentity)
        // }
        pc.onconnectionstatechange = this.handleConnectionStatus(peerIdentity);
        if (this.main.srcObject) {
          this.main.srcObject.getTracks().forEach(track => {
            if (this.main.srcObject) pc.addTrack(track, this.main.srcObject);
          });
        }
        const peerData = {
          srcObject: null,
          phid,
          peerIdentity,
          pc,
          status: "new"
        };
        if (phid === this.main.phid) {
          this.main.pc = pc;
        } else {
          this.child.push(peerData);
        }
        if (isInitiator) {
          await this.createOffer(peerIdentity);
        }
      } catch (e) {
        console.log("error on creating peer", e);
      }
    },
    handleRemoteStream(peerIdentity: string) {
      return (event: RTCTrackEvent) => {
        if (event.streams.length && this.main.peerIdentity !== peerIdentity) {
          console.log("got remote stream", event)
          const index = this.findIdx(peerIdentity)
          let remoteStream: MediaStream | null = null;
          if (this.child[index].srcObject) {
            remoteStream = this.child[index].srcObject
          } else {
            remoteStream = new MediaStream()
          }
          event.streams[0].getTracks().forEach((track: MediaStreamTrack) => {
            if (track.kind === "video")
              console.log("got video", track)
            track.onended = () => {
              console.log(`this ${track} is ended`)
            }
            remoteStream?.addTrack(track);
          });
          this.child[index].srcObject = null;
          this.child[index].srcObject = remoteStream;
        }
      };
    },
    handleIceCandidate(peerIdentity: string) {
      return (event: any) => {
        if (event.candidate) {
          const iceCandidate = event.candidate;
          const data = {
            candidate: iceCandidate.candidate,
            id: iceCandidate.sdpMid,
            label: iceCandidate.sdpMLineIndex
          };
          this.sendMessage("candidate", data, peerIdentity);
        }
      };
    },
    handleConnectionStatus(peerIdentity: string) {
      return (ev: any) => {
        if (ev.currentTarget.connectionState) {
          const connectionState = ev.currentTarget.connectionState;
          console.log(connectionState, peerIdentity)
          const index = this.findIdx(peerIdentity)
          this.child[index].status = connectionState;
        }
      };
    },
    async createOffer(peerIdentity: string) {
      const offerOptions = {
        offerToReceiveAudio: true,
        offerToReceiveVideo: true
      };
      console.log(`creating offer for ${peerIdentity}`);
        const index = this.findIdx(peerIdentity)
        const sessionDescription = await this.child[index]?.pc?.createOffer(offerOptions);
        if (sessionDescription) {
          await this.child[index]?.pc?.setLocalDescription(sessionDescription);
          this.sendMessage("offer", sessionDescription, peerIdentity);
        }
    },
    async handleOffer(payload: {
      from: string;
      data: RTCSessionDescriptionInit;
    }) {
      let index = this.findIdx(payload.from)
      if (index === -1) {
        await this.initRTC(payload.from, "", false);
        index = this.findIdx(payload.from)
      }
      await this.child[index].pc?.setRemoteDescription(
          new RTCSessionDescription(payload.data)
      );
      const sessionDescription = await this.child[index].pc?.createAnswer();
      if (sessionDescription) {
        await this.child[index]?.pc?.setLocalDescription(sessionDescription);
        this.sendMessage("answer", sessionDescription, payload.from);
      }
    },
    handleAnswer(payload: { from: string; data: RTCSessionDescriptionInit }) {
      const index = this.findIdx(payload.from)
      if (index > -1)
        this.child[index].pc?.setRemoteDescription(
            new RTCSessionDescription(payload.data)
        );
    },
    handleCandidate(payload: {
      data: { label: number; candidate: string; id: string };
      from: string;
    }) {
      const index = this.findIdx(payload.from)
      const candidate = new RTCIceCandidate({
        sdpMLineIndex: payload.data.label,
        candidate: payload.data.candidate,
        sdpMid: payload.data.id
      });
      if (index > -1) this.child[index].pc?.addIceCandidate(candidate);
    },
    async handleJoined(payload: {
      data: { peer_id: string; username: string };
    }) {
      const { peer_id: peerIdentity, username } = payload.data;
      const index = this.findIdx(peerIdentity)
      if (this.child[index]) {
        this.child[index].phid = username;
      }
      await this.initRTC(peerIdentity, username, true);
    },
    handleLeave(payload: { data: { peer_id: string } }) {
      const { peer_id: peerIdentity } = payload.data;
      const index = this.findIdx(peerIdentity)
      this.child[index].pc?.close();
      this.child[index].srcObject?.getTracks().forEach(track => {
        track.stop();
      });
      this.child[index].srcObject = null;
      this.child.splice(index, 1);
    },
    handleMic() {
      if (this.main.srcObject) {
        this.main.srcObject.getAudioTracks()[0].enabled = !this.main.srcObject.getAudioTracks()[0]
            .enabled;
        this.main.enabled = this.main.srcObject.getAudioTracks()[0].enabled
      }
    },
    async handleShare() {
      const mediaDevices = navigator.mediaDevices as any
      const screenStream = await mediaDevices.getDisplayMedia()
      const videoStream = screenStream.getVideoTracks()[0]
      videoStream.onended = async () => {
        console.log(`this ${videoStream} is ended`)
        this.main.srcObject?.removeTrack(this.main.srcObject?.getAudioTracks()[0])
        if (this.share)
          this.main.pc?.removeTrack(this.share)
      }
      this.main.srcObject?.addTrack(videoStream)
      if (this.main.srcObject)
        this.share = this.main.pc?.addTrack(videoStream, this.main.srcObject)
      for (const ch of this.child) {
        console.log("ch", ch)
        await this.createOffer(ch.peerIdentity)
      }
    }
  },
  // watch: {
  //   child: {
  //     deep: true,
  //     handler(val: ERtcData[]) {
  //       console.log(val[0].srcObject?.getTracks())
  //     }
  //   }
  // }
})
</script>