<template>
  <van-popup
    :value="showPopup"
    position="bottom"
    :style="{ height: '95%' }"
  />
</template>

<script>
export default {
  name: 'HomeChannel',
  props: {
    showPopup: {
      type: 'boolean',
      required: true
    }
  },
  data () {
    return {
      show: true
    }
  }
}
</script>

<style lang="less" scoped>

</style>
