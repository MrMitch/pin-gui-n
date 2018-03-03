<template>
  <div class="pin" :class="{ 'd-none': hidden }" :style="style">
    <span>{{ coordinate(this.x) }}&thinsp;;&thinsp;{{ coordinate(this.y) }}</span>
  </div>
</template>

<script>
export default {
  name: 'pin',
  props: {
    x: Number,
    y: Number,
    originX: {
      required: false,
      default: 0
    },
    originY: {
      required: false,
      default: 0
    },
    hidden: false,
    unit: Object
  },
  data () {
    return {
      removing: false
    }
  },
  computed: {
    style () {
      return { top: `${this.y}px`, left: `${this.x}px` }
    }
  },
  methods: {
    coordinate (px) {
      return `${Math.round(px * this.unit.factor, 2)}`
    }
  }
}
</script>

<style lang="scss">
.pin {
  position: absolute;
  line-height: 20px;
  height: 20px;
  border-top: 1px solid black;
  border-left: 1px solid black;
  background: rgba(230, 230, 230, 0.5);
  box-sizing: border-box;
  font-family: 'Fira Code', 'Ubuntu Code', 'Consolas', 'Monaco', 'Courier', 'Monospace', 'sans-serif';
  user-select: none;
  cursor: move;
}
</style>
