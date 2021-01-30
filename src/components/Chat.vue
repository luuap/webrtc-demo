<template>
  <div class="hello">
    <button @click="createRoom" :disabled="createBtnDisabled">Create Room</button><br>
    <input ref="roomInput" :disabled="roomInputDisabled">
    <button @click="joinRoom" :disabled="joinBtnDisabled">Join Room</button><br>
    <button @click="disconnect" :disabled="disconnectBtnDisabled">Disonnect</button><br>
    <ul>
      <li v-for="(message, index) in messages" :key="index">
        {{ message }}
      </li>
    </ul>
    <input ref="messageInput" :disabled="messageInputDisabled">
    <button @click="sendMessage" :disabled="sendBtnDisabled">Send</button>
  </div>
</template>

<script lang="ts">
import { Vue } from 'vue-class-component';

export default class HelloWorld extends Vue {

  private DEBUG = true;
  private STUN_CONFIG = {
    iceServers: [
      {
        urls: [
          'stun:stun1.l.google.com:19302',
          'stun:stun2.l.google.com:19302',
        ],
      },
    ],
    iceCandidatePoolSize: 10,
  };
  private SIGNALLING_URL = process.env.VUE_APP_SIGNALLING_URL;

  $refs!: {
    messageInput: HTMLInputElement,
    roomInput: HTMLInputElement,
  }

  peerConnection: RTCPeerConnection | null = null;
  signallingConnection: WebSocket | null = null;
  dataChannel: RTCDataChannel | null = null;

  createBtnDisabled = false;
  roomInputDisabled = false;
  joinBtnDisabled = false;
  disconnectBtnDisabled = true;
  sendBtnDisabled = true;
  messageInputDisabled = true;

  roomId: string | null = null;
  messages: string[] = ['Messages'];

  // temporarily store candidates if we get them before we call setRemoteDescription
  candidateQueue: RTCIceCandidateInit[] = [];

  createRoom() {
    // ui logic
    this.createBtnDisabled = true;
    this.joinBtnDisabled = true;
    this.roomInputDisabled = true;
    this.disconnectBtnDisabled = false;

    // connect to signalling server
    this.signallingConnection = new WebSocket(this.SIGNALLING_URL);
    this.signallingConnection.onopen = async () => {
      console.log('[SignallingServer] Connected to signalling server');

      this.peerConnection = new RTCPeerConnection(this.STUN_CONFIG);
      this.setPeerConnectionListeners();

      // create data channel and set listeners
      this.dataChannel = this.peerConnection.createDataChannel("chat");
      this.setdataChannelListeners();

      // create offer and send to signalling server
      const offer = await this.peerConnection.createOffer();
      await this.peerConnection.setLocalDescription(offer); // this will trigger ice candidates
      console.log('[PeerConnection] Created offer:', JSON.stringify({action: 'addOffer', offer}));
      this.signallingConnection?.send(JSON.stringify({action: 'addOffer', offer}));

    };

    // listen to messages from signalling server
    this.signallingConnection.onmessage = this.handleSignallingMessages;
    this.signallingConnection.onclose = () => {
      console.log('Disconnected from signalling server');
    }
    // TODO check if we can connect to signalling server
  }

  joinRoom() {
    
    this.roomId = this.$refs.roomInput.value;
    // TODO validation

    // ui logic
    this.createBtnDisabled = true;
    this.joinBtnDisabled = true;
    this.roomInputDisabled = true;
    this.disconnectBtnDisabled = false;

    // connect to signalling server
    this.signallingConnection = new WebSocket(this.SIGNALLING_URL);
    this.signallingConnection.onopen = () => {
      console.log('[SignallingServer] Connected to signalling server');

      this.peerConnection = new RTCPeerConnection(this.STUN_CONFIG);
      this.setPeerConnectionListeners();

      // set data channel listeners
      this.peerConnection.ondatachannel = (event) => {
        this.dataChannel = event.channel;
        this.setdataChannelListeners();
      };

      // trigger joinRoom route
      this.signallingConnection?.send(JSON.stringify({action: 'joinRoom', roomId: this.roomId}));
    }

    this.signallingConnection.onmessage = this.handleSignallingMessages;
    this.signallingConnection.onclose = () => {
      console.log('Disconnected from signalling server');
    }
  }

  sendMessage() {
    const message = this.$refs.messageInput.value;
    this.dataChannel?.send(message);
    this.messages.push('Me: ' + message);
  }

  disconnect() {

    // clean up
    if (this.signallingConnection?.readyState !== WebSocket.CLOSED) {
      this.signallingConnection!.send(JSON.stringify({action: '$disconnect'}));
      this.signallingConnection!.close();
    };

    this.dataChannel?.close();
    this.peerConnection?.close();

    this.signallingConnection = null;
    this.peerConnection = null;
    this.dataChannel = null;

    // ui logic
    this.createBtnDisabled = false;
    this.joinBtnDisabled = false;
    this.roomInputDisabled = false;
    this.disconnectBtnDisabled = true;
    this.sendBtnDisabled = true;
    this.messageInputDisabled = true;
  }

  private async handleSignallingMessages(event: MessageEvent) {
    console.log('Got message: ', event.data);
    const data: Record<string, any>[] = JSON.parse(event.data);

    if (!Array.isArray(data)) return;
    const tasks = data.map(async (d) => {
      switch(d.action) {
        case 'addOffer': {
          const rtcSessionDescription = new RTCSessionDescription(d.offer);
          await this.peerConnection?.setRemoteDescription(rtcSessionDescription);

          // deal with queue
          await this.consumeCandidateQueue();

          // create answer
          const answer = await this.peerConnection!.createAnswer();
          await this.peerConnection?.setLocalDescription(answer);
          
          // send answer
          this.signallingConnection?.send(JSON.stringify({action: 'addAnswer', answer, roomId: this.roomId}));
          console.log('[PeerConnection] Created answer:', JSON.stringify(JSON.stringify({action: 'addAnswer', answer, roomId: this.roomId})));

          break;
        }
        case 'sendRoomId': {
          this.roomId = d.roomId;
          this.messages.push(`The roomId is ${d.roomId}`);
          break;
        }
        case 'addAnswer': {
          const rtcSessionDescription = new RTCSessionDescription(d.answer);
          await this.peerConnection?.setRemoteDescription(rtcSessionDescription);

          // deal with queue
          await this.consumeCandidateQueue();
          break;
        }
        case 'addIceCandidate': {
          if (!this.peerConnection?.remoteDescription) { // remote description must be set before candidates are added
            // add to queue
            this.candidateQueue.push(d.candidate);
          } else {
            await this.peerConnection?.addIceCandidate(new RTCIceCandidate(d.candidate));
            console.log('Added ice candidate: ', d.candidate);
          }
          break;
        }
        case 'informAvailableAnswer': {
          // this will trigger the answer and candidate to be sent
          this.signallingConnection?.send(JSON.stringify({action: 'receiveAnswer', roomId: this.roomId}));
        }
      }
    });
    await Promise.all(tasks);
  }

  private setPeerConnectionListeners() { // peerConnection must be set before calling this

    // set callback for some state changes
    if (this.DEBUG === true) {
      this.peerConnection!.oniceconnectionstatechange = () => {
        console.log('[PeerConnection] ConnectionState: ', this.peerConnection!.iceConnectionState)
      };
      this.peerConnection!.onnegotiationneeded = () => {
        console.log('[PeerConnection] NegotiationNeeded');
      };
      this.peerConnection!.onicegatheringstatechange = () => {
        console.log('[PeerConnection] IceGatheringState: ', this.peerConnection!.iceGatheringState)
      };
      this.peerConnection!.onsignalingstatechange = () => {
        console.log('[PeerConnection] SignallingState: ', this.peerConnection!.signalingState)
      };
      this.peerConnection!.onicecandidateerror = (error) => {
        console.error('[PeerConnection] Error', error);
      };
    };

    // set callback when peer gathers an ice candidate
    this.peerConnection!.onicecandidate = (event) => {
      if (!event.candidate) {
        console.log('[PeerConnection] Got final candidate!')
      } else {
        console.log('[PeerConnection] Got candidate: ', JSON.stringify({candidate: event.candidate.toJSON()}));
        // send Ice candidate to callee
        this.signallingConnection?.send(JSON.stringify({action: 'addIceCandidate', candidate: event.candidate.toJSON()}));
      };
    };

  }

  private setdataChannelListeners() { // message channel must be set before calling  
    this.dataChannel!.onopen = () => {
      console.log('[DataChannel] Ready to send messages');
      this.sendBtnDisabled = false;
      this.messageInputDisabled = false;
      this.signallingConnection?.send(JSON.stringify({action: '$disconnect'}));
      this.signallingConnection?.close();
    };
    this.dataChannel!.onmessage = (event) => {
      this.messages.push('Other: ' + event.data);
    };
    this.dataChannel!.onerror = (error) => {
      console.error('[DataChannel] Error: ', error);
    };
    this.dataChannel!.onclose = () => {
      console.log('[DataChannel] dataChannel Closed');
    };
  }

  private async consumeCandidateQueue() {
    console.log('length of queue', this.candidateQueue.length);
    const tasksQueue = this.candidateQueue.map(async (val) => {
      await this.peerConnection?.addIceCandidate(new RTCIceCandidate(val));
      console.log('Added ice candidate: ', val);
    });
    await Promise.all(tasksQueue);
    this.candidateQueue = [];
  }

}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
