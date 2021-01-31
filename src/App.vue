<template>
  <h1>RTC-DEMO</h1>
  <AppLobby
    v-if="lobbyVisible"
    @create-room="createRoom"
    @join-room="joinRoom">
  </AppLobby>
  <AppCountdown
    v-else-if="countdownVisible"
    @finished="onCountdownFinished"
    :timeout="60"
  >
    <h3>Room ID is: {{ roomId }}</h3>
    <h4>Send this ID before the timer runs out!</h4>
  </AppCountdown>
  <AppChat 
    v-else-if="chatVisible" 
    :messages="messages" 
    @send-message="sendMessage"
    @disconnect="disconnect">
  </AppChat>
</template>

<script lang="ts">
import { Vue, Options } from "vue-class-component";

import AppChat from "./components/AppChat.vue";
import AppCountdown from "./components/AppCountdown.vue";
import AppLobby from "./components/AppLobby.vue";

@Options({
  components: {
    AppChat,
    AppCountdown,
    AppLobby,
  }
})
export default class App extends Vue {
  private DEBUG = true;
  private STUN_CONFIG = {
    iceServers: [
      {
        urls: ["stun:stun1.l.google.com:19302", "stun:stun2.l.google.com:19302"]
      }
    ],
    iceCandidatePoolSize: 10
  };
  private SIGNALLING_URL = process.env.VUE_APP_SIGNALLING_URL;

  peerConnection: RTCPeerConnection | null = null;
  signallingConnection: WebSocket | null = null;
  dataChannel: RTCDataChannel | null = null;

  roomId: string | null = null;

  // ui states
  lobbyVisible = true;
  countdownVisible = false;
  chatVisible = false;

  // temporarily store candidates if we get them before we call setRemoteDescription
  candidateQueue: RTCIceCandidateInit[] = [];

  messages: string[] = ['Messages'];

  createRoom() {
    // connect to signalling server
    this.signallingConnection = new WebSocket(this.SIGNALLING_URL);
    this.signallingConnection.onopen = async () => {
      console.log("[SignallingServer] Connected to signalling server");

      this.peerConnection = new RTCPeerConnection(this.STUN_CONFIG);
      this.setPeerConnectionListeners();

      // create data channel and set listeners
      this.dataChannel = this.peerConnection.createDataChannel("chat");
      this.setdataChannelListeners();

      // create offer and send to signalling server
      const offer = await this.peerConnection.createOffer();
      await this.peerConnection.setLocalDescription(offer); // this will trigger ice candidates
      console.log(
        "[PeerConnection] Created offer:",
        JSON.stringify({ action: "addOffer", offer })
      );
      this.signallingConnection?.send(
        JSON.stringify({ action: "addOffer", offer })
      );
    };

    // listen to messages from signalling server
    this.signallingConnection.onmessage = this.handleSignallingMessages;
    this.signallingConnection.onclose = () => {
      console.log("Disconnected from signalling server");
    };
    // TODO check if we can connect to signalling server
  }

  onCountdownFinished() {
    console.log("Countdown finished");
    this.disconnect();
    this.lobbyVisible = true; // TODO check if component is refreshed
    this.countdownVisible = false;
  }

  joinRoom(roomId: string) {
    this.roomId = roomId;

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

  sendMessage(message: string) {
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
    this.lobbyVisible = true; // TODO check if component is refreshed
    this.chatVisible = false;
  }

  beforeUnmount() {
    this.disconnect();
  }

  private async handleSignallingMessages(event: MessageEvent) {
    console.log("Got message: ", event.data);
    const data: Record<string, any>[] = JSON.parse(event.data);

    if (!Array.isArray(data)) return;
    const tasks = data.map(async d => {
      switch (d.action) {
        case "addOffer": {
          const rtcSessionDescription = new RTCSessionDescription(d.offer);
          await this.peerConnection?.setRemoteDescription(
            rtcSessionDescription
          );

          // deal with queue
          await this.consumeCandidateQueue();

          // create answer
          const answer = await this.peerConnection!.createAnswer();
          await this.peerConnection?.setLocalDescription(answer);

          // send answer
          this.signallingConnection?.send(
            JSON.stringify({ action: "addAnswer", answer, roomId: this.roomId })
          );
          console.log(
            "[PeerConnection] Created answer:",
            JSON.stringify(
              JSON.stringify({
                action: "addAnswer",
                answer,
                roomId: this.roomId
              })
            )
          );

          break;
        }
        case "sendRoomId": {

          // ui logic
          this.lobbyVisible = false;
          this.countdownVisible = true;

          this.roomId = d.roomId;
          break;
        }
        case "addAnswer": {
          const rtcSessionDescription = new RTCSessionDescription(d.answer);
          await this.peerConnection?.setRemoteDescription(
            rtcSessionDescription
          );

          // deal with queue
          await this.consumeCandidateQueue();
          break;
        }
        case "addIceCandidate": {
          if (!this.peerConnection?.remoteDescription) {
            // remote description must be set before candidates are added
            // add to queue
            this.candidateQueue.push(d.candidate);
          } else {
            await this.peerConnection?.addIceCandidate(
              new RTCIceCandidate(d.candidate)
            );
            console.log("Added ice candidate: ", d.candidate);
          }
          break;
        }
        case "informAvailableAnswer": {
          // this will trigger the answer and candidate to be sent
          this.signallingConnection?.send(
            JSON.stringify({ action: "receiveAnswer", roomId: this.roomId })
          );
        }
      }
    });
    await Promise.all(tasks);
  }

  // peerConnection must be set before calling this
  private setPeerConnectionListeners() {

    // set callback for some state changes
    if (this.DEBUG === true) {
      this.peerConnection!.oniceconnectionstatechange = () => {
        console.log(
          "[PeerConnection] ConnectionState: ",
          this.peerConnection!.iceConnectionState
        );
      };
      this.peerConnection!.onnegotiationneeded = () => {
        console.log("[PeerConnection] NegotiationNeeded");
      };
      this.peerConnection!.onicegatheringstatechange = () => {
        console.log(
          "[PeerConnection] IceGatheringState: ",
          this.peerConnection!.iceGatheringState
        );
      };
      this.peerConnection!.onsignalingstatechange = () => {
        console.log(
          "[PeerConnection] SignallingState: ",
          this.peerConnection!.signalingState
        );
      };
      this.peerConnection!.onicecandidateerror = error => {
        console.error("[PeerConnection] Error", error);
      };
    }

    // set callback when peer gathers an ice candidate
    this.peerConnection!.onicecandidate = event => {
      if (!event.candidate) {
        console.log("[PeerConnection] Got final candidate!");
      } else {
        console.log(
          "[PeerConnection] Got candidate: ",
          JSON.stringify({ candidate: event.candidate.toJSON() })
        );
        // send Ice candidate to callee
        this.signallingConnection?.send(
          JSON.stringify({
            action: "addIceCandidate",
            candidate: event.candidate.toJSON()
          })
        );
      }
    };
  }

  // message channel must be set before calling this
  private setdataChannelListeners() {
    this.dataChannel!.onopen = () => {
      console.log("[DataChannel] Ready to send messages");

      // set ui state
      this.lobbyVisible = false;
      this.countdownVisible = false;
      this.chatVisible = true;

      this.signallingConnection?.send(
        JSON.stringify({ action: "$disconnect" })
      );
      this.signallingConnection?.close();
    };
    this.dataChannel!.onmessage = event => {
      this.messages.push("Other: " + event.data);
    };
    this.dataChannel!.onerror = error => {
      console.error("[DataChannel] Error: ", error);
    };
    this.dataChannel!.onclose = () => {
      console.log("[DataChannel] dataChannel Closed");
    };
  }

  private async consumeCandidateQueue() {
    console.log("length of queue", this.candidateQueue.length);
    const tasksQueue = this.candidateQueue.map(async val => {
      await this.peerConnection?.addIceCandidate(new RTCIceCandidate(val));
      console.log("Added ice candidate: ", val);
    });
    await Promise.all(tasksQueue);
    this.candidateQueue = [];
  }
}
</script>

<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
