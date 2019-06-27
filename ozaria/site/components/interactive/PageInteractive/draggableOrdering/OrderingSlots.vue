<script>

  export default {
    props: {
      labels: {
        type: Array,
        default: () => ([])
      }
    },
  }
</script>

<template>
  <div class="ordering-slots-container">
    <ul
      v-for="label in labels"
      :key="label"

      class="draggable-slot"

      @change="changed"
    >
      <li>{{ label.text }}</li>
    </ul>
  </div>
</template>

<style scoped lang="scss">
  .ordering-slots-container {
    width: 25%;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: space-evenly;
    /deep/ .draggable-slot {
      height: 55px;
      background-color: #dddddd;
      padding: 0;
      width: 100%;
      display: flex;
      li {
        margin: 0 auto;
      }
    }
  }
  .ordering-slot li {
    width: 100%;
    height: 100%;
  }
</style>