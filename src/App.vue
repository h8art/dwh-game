<template lang='pug'>
  #app
    game
</template>

<script>
import Game from './components/Game.vue'

export default {
  name: 'app',
  components: {
    Game
  }
}
</script>

<style lang='sass'>
body
  margin: 0
</style>
