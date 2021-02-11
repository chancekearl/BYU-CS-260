<template>
  <div class="home">
    <image-gallery :photos="photos" />
    <p v-if="error">{{error}}</p>

  </div>
</template>

<script>
import axios from 'axios';
import ImageGallery from '@/components/ImageGallery.vue';
export default {
  name: 'Home',
  components: {
    ImageGallery,
  },
  data() {
    return {
      photos: [],
      error: '',
    }
  },
  created() {
    this.getPhotos();
  },
  methods: {
    async getPhotos() {
      try {
        let response = await axios.get("/api/photos/all");
        this.photos = response.data;
      } catch (error) {
        this.error = error.response.data.message;
      }
    },
  },
}
</script>