<template>
  <div class="main">
  <h1 id="title">{{ title }}</h1>
  <label for="url">シグナリングサーバのURL:</label>
  <input type="text" id="url" name="url" required value="ws://localhost:3000/ws">
  <div id="buttons">
  <button type="button" @click="connect()">接続</button>
  <button type="button" @click="disconnect">切断</button>
  </div>
  <div id="videos">
  <video ref="remoteVideo" id="remote-video" autoplay></video>
  <video ref="localVideo" id="local-video" autoplay></video>
  </div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';

@Component
export default class P2P extends Vue {
  @Prop() private title!: string;
  private wsUrl: string = 'ws://localhost:3000/ws';
  private ws: WebSocket | null = null;
  private isNegotiating: boolean = false;
  private peerConnection: RTCPeerConnection | null = null;
  private localVideo: any = null;
  private localStream: MediaStream | null = null;
  private peerConnectionConfig: object = {
    iceServers: [{ urls: 'stun:stun.l.google.com:19302' }],
  };
  private tracks: any[] = [];

  public mounted(): void {
    this.startVideo();
  }


  public connect(): void {
    this.tracks = [];
    this.isNegotiating = false;
    // ws がない場合は新規作成
    this.ws = new WebSocket(this.wsUrl);
    this.ws.onmessage = this.onWsMessage.bind(this);

    if (!this.peerConnection) {
      this.prepareNewConnection(true);
      if (this.localStream) {
        const videoTrack = this.localStream.getVideoTracks()[0];
        const audioTrack = this.localStream.getAudioTracks()[0];
        if (this.peerConnection) {
          if (videoTrack) {
            this.peerConnection.addTrack(videoTrack, this.localStream);
          }
          if (audioTrack) {
            this.peerConnection.addTrack(audioTrack, this.localStream);
          }
        }
      } else {
        console.warn('no local stream, but continue.');
      }
    } else {
      console.warn('peer already exist.');
    }
  }

  public disconnect(): void {
    if (this.peerConnection) {
      if (this.peerConnection.iceConnectionState !== 'closed') {
        // peer connection を閉じる
        this.peerConnection.close();
        const message = JSON.stringify({ type: 'close' });
        console.log('sending close message');
        if (this.ws) {
          this.ws.send(message);
        } else {
          console.error('websocket connection does not exist!');
        }
        this.peerConnection = null;
      }
    }
  }

  private async startVideo(): Promise<void> {
    try {
      this.localStream = await navigator.mediaDevices.getUserMedia({video: true, audio: true});
      this.playLocalVideo();
    } catch (error) {
      console.error('mediaDevice.getUserMedia() error:', error);
    }
  }

  private playLocalVideo() {
    const video: HTMLVideoElement = this.$refs.localVideo as HTMLVideoElement;
    video.srcObject = this.localStream;
    console.log('play local video');
    video.play();
  }

  private playRemoteVideo(remoteStream: MediaStream) {
    const video: HTMLVideoElement = this.$refs.remoteVideo as HTMLVideoElement;
    video.srcObject = remoteStream;
    console.log('play remote video');
    video.play();
  }

  private async setAnswer(sessionDescription: RTCSessionDescription) {
    try {
      await this.peerConnection.setRemoteDescription(sessionDescription);
    } catch (error) {
      console.error('setRemoteDescription(answer) ERROR: ', error);
    }
  }

  private async makeAnswer() {
    try {
      const answer = await this.peerConnection.createAnswer();
      await this.peerConnection.setLocalDescription(answer);
      const localDescription = this.peerConnection.localDescription;
      this.sendSdp(localDescription);
    } catch (error) {
      console.error('makeAnswer ERROR: ', error);
    }
  }

  private async setOffer(sessionDescription: RTCSessionDescription) {
    this.prepareNewConnection(false);
    try {
      if (this.peerConnection) {
        await this.peerConnection.setRemoteDescription(sessionDescription);
        console.log('setRemoteDescription(answer) success in promise');
        this.makeAnswer();
      }
    } catch (error) {
      console.error('setRemoteDescription(offer) ERROR: ', error);
    }
  }

  private sendSdp(sessionDescription: RTCSessionDescription) {
    if (this.ws) {
      console.log('---sending sdp ---');
      const message = JSON.stringify(sessionDescription);
      console.log('sending SDP=' + message);
      this.ws.send(message);
    } else {
      console.error('websocket connection does not exist!');
    }
  }

  private prepareNewConnection(isOffer: boolean): void {
    const peer: RTCPeerConnection = new RTCPeerConnection(this.peerConnectionConfig);

    if ('ontrack' in peer) {
      peer.ontrack = this.onTrack.bind(this);
    } else {
      peer.onaddstream = this.onAddStream.bind(this);
    }

    peer.onicecandidate = (event: RTCPeerConnectionIceEvent) => {
      if (event.candidate) {
        console.log('-- peer.onicecandidate()', event.candidate);
        const candidate = event.candidate;
        const message = JSON.stringify({ type: 'candidate', ice: candidate });
        if (this.ws) {
          this.ws.send(message);
        }
      } else {
        console.log('empty ice event');
      }
    };

    peer.onnegotiationneeded = async () => {
      if (this.isNegotiating) {
        return;
      }
      try {
        this.isNegotiating = true;
        if (isOffer) {
          const offer = await peer.createOffer({ offerToReceiveAudio: true, offerToReceiveVideo: true});
          await peer.setLocalDescription(offer);
          this.sendSdp(peer.localDescription);
        }
        this.isNegotiating = false;
      } catch (error) {
        console.error('setLocalDescription(offer) ERROR: ', error);
      }
    };

    peer.oniceconnectionstatechange = () => {
      switch (peer.iceConnectionState) {
        case 'connected':
          this.isNegotiating = false;
          break;
        case 'closed':
        case 'failed':
          if (this.peerConnection) {
            this.disconnect();
          }
          break;
        case 'disconnected':
          break;
      }
    };
    this.peerConnection = peer;
  }

  private onAddStream(event: MediaStreamEvent): void {
    console.log('-- peer.onaddstream()', event);
    if (event.stream) {
      const stream = event.stream as MediaStream;
      this.playRemoteVideo(stream);
    }
  }

  private onTrack(event: RTCTrackEvent): void {
    console.log('-- peer.ontrack()', event);
    this.tracks.push(event.track);
    const mediaStream = new MediaStream(this.tracks);
    if (mediaStream) {
      this.playRemoteVideo(mediaStream);
    }
  }

  private onWsMessage(event: MessageEvent): void {
    console.log('ws onmessage() data:', event.data);
    const message = JSON.parse(event.data);
    switch (message.type) {
      case 'offer':
        console.log('Received offer ...', message);
        this.setOffer(message);
        break;
      case 'answer':
        console.log('Received answer ...', message);
        this.setAnswer(message);
        break;
      case 'candidate':
        console.log('Received ICE candidate ...');
        if (this.peerConnection) {
          const candidate: RTCIceCandidate = new RTCIceCandidate(message.ice);
          this.peerConnection.addIceCandidate(candidate);
        }
        break;
      case 'close':
        console.log('peer is closed ...');
        this.disconnect();
        break;
      default:
        console.log('Invalid message type: ', message.type);
        break;
    }
  }

}
</script>

<style scoped>
#title {
  position: absolute;
  z-index: 3;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  top: 40px;
}
#videos {
  font-size: 0;
  pointer-events: none;
  position: absolute;
  transition: all 1s;
  width: 100%;
  height: 100%;
  display: block;
}

#buttons {
  position: absolute;
  z-index: 3;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  top: 120px;
}

#remote-video {
  height: 100%;
  max-height: 100%;
  max-width: 100%;
  object-fit: cover;
  transform: scale(-1, 1);
  transition: opacity 1s;
  width: 100%;
}

#local-video {
  z-index: 2;
  border: 1px solid gray;
  bottom: 20px;
  right: 20px;
  max-height: 17%;
  max-width: 17%;
  position: absolute;
  transition: opacity 1s;
}
</style>
