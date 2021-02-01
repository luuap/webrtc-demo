<template>
<div class="app-chat">
  <h3>You are in room #{{ roomId }}</h3>
  <ul class="messages">
    <li v-for="(message, index) in messages" :key="index">
      {{ message }}
    </li>
  </ul>
  <input ref="messageInput">
  <button @click="sendMessage">Send</button>
  <button @click="disconnect">Disonnect</button>
</div>
</template>

<script lang="ts">
import { Vue, Options, prop } from 'vue-class-component';

class Props {
  messages = prop({
    type: Array,
    required: true,
  });
  roomId = prop({
    type: String,
    required: true,
  });
}

@Options({
  emits: ['sendMessage', 'disconnect'],
})
export default class AppChat extends Vue.with(Props) {

  declare $refs: {
    messageInput: HTMLInputElement,
  }

  sendMessage() {
    this.$emit('sendMessage', this.$refs.messageInput.value);
    this.$refs.messageInput.value = '';
  }

  disconnect() {
    this.$emit('disconnect');
  }
}
</script>

<style scoped lang="scss">
.messages {
  list-style-type: none;
  padding: 0;

  & > {
    margin: 0 10px;
  }

}
.app-chat {
  display: flex;
  flex-direction: column;
  align-items: center;

  & button {
    width: 130px;
    margin: 0.8em 0 0 0;
  }

}
</style>