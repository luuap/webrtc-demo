<template>
  <slot></slot>
  <h3>{{ time }}</h3>
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
  time: number = this.timeout;

  private interval: number | null= null;

  mounted() {
    this.interval = setInterval(() => {
      this.time -= 1;
      if (this.time === 0) {
        clearInterval(this.interval!);
        this.$emit('finished');
      }
    }, 1000);
  }
  
  beforeUnmount() {
    clearInterval(this.interval!);
  }
}
</script>
