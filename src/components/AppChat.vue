<template>
  <h3>Messages</h3>
  <ul class="messages">
    <li v-for="(message, index) in messages" :key="index">
      {{ message }}
    </li>
  </ul>
  <input ref="messageInput">
  <button @click="sendMessage">Send</button>
  <button @click="disconnect">Disonnect</button>
</template>

<script lang="ts">
import { Vue, Options, prop } from 'vue-class-component';

class Props {
  messages = prop({
    type: Array,
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
</style>