<template>
  <slot></slot>
  <h3>{{ remainingSeconds }}</h3>
</template>

<script lang="ts">
import { Vue, Options, prop } from 'vue-class-component';

class Props {
  timeout = prop({
    type: Number,
    required: true,
    validator: (value: number) => value >= 0
  });
}

@Options({
  emits: ['finished'],
})
export default class AppCountdown extends Vue.with(Props) {
  remainingSeconds: number = this.timeout;

  private timer: number | null = null;

  private startTime = performance.now();

  mounted() {
    this.tick(this.startTime);
  }
  
  beforeUnmount() {
    clearTimeout(this.timer!);
  }

  /**
   * Minimizes drift https://gist.github.com/jakearchibald/cb03f15670817001b1157e62a076fe95
   */
  private tick(timestamp: number) {

    const elapsed = timestamp - this.startTime;
    const elapsedSeconds = Math.round(elapsed / 1000);
    const targetNext = this.startTime + (elapsedSeconds * 1000) + 1000;
    const delay = targetNext - performance.now();

    this.remainingSeconds = this.timeout - elapsedSeconds;

    if (this.remainingSeconds <= 0) {
      this.$emit('finished');
      return;
    }

    this.timer = setTimeout(() => requestAnimationFrame(this.tick), delay);
  }
}
</script>
