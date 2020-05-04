<template>
  <div class="app-container">
    <Upload :path="path" />
  </div>
</template>

<script>
import Upload from '@/components/Uploader/Upload'

export default {
  components: {
    Upload
  },
  data() {
    return {
      files: [],
      path: ''
    }
  },
  methods: {

  }
}
</script>

<style scoped>

</style>
