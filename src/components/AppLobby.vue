<template>
  <div v-if="!loading" class="app-lobby">
    <div>
      <button @click="createRoom">Create Room</button>
    </div>
    <div>
      <input ref="roomIdInput">
      <button @click="joinRoom">Join Room</button>
    </div>
  </div>
  <h3 v-if="loading">Loading...</h3>
</template>

<script lang="ts">
import { Vue, Options } from 'vue-class-component';

@Options({
  emits: ['joinRoom', 'createRoom']
})
export default class AppLobby extends Vue {

  loading = false;

  declare $refs: {
    roomIdInput: HTMLInputElement,
  }

  createRoom() {
    this.loading = true;
    this.$emit('createRoom');
  }

  joinRoom() {
    const roomId = this.$refs.roomIdInput.value.trim();
    this.loading = true;
    if (roomId !== '' ) {
      this.$emit('joinRoom', roomId);
    }
  }

}
</script>

<style scoped lang="scss">
.app-lobby {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  grid-gap: 1rem;
}
</style>