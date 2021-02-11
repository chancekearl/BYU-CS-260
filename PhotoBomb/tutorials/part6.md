# Part 6: Home Page

The last step is to display everyone's photos on the home page.

## Back End

On the back end, we need an API endpoint that will return all the photos in
the system. In `back-end/photos.js`, add the following endpoint:

```javascript
// get all photos
router.get("/all", async (req, res) => {
  try {
    let photos = await Photo.find().sort({
      created: -1
    }).populate('user');
    return res.send(photos);
  } catch (error) {
    console.log(error);
    return res.sendStatus(500);
  }
});
```

We have to use a different path for this endpoint, since we already have a GET
endpoint for getting the photos for the current user. We do _not_ validate the
token or the user's account because anyone can see any photos.

For this query, we again add `populate('user')` to have Mongo fill in the user
information for each photo. This way we can use `photo.user.name` to get the
name of the person who took the photo.

## Front End

We need to modify `Home.vue` to show photos uploaded from all users. First, modify the `template`:

```html
<template>
<div class="home">
  <image-gallery :photos="photos" />
  <p v-if="error">{{error}}</p>
</div>
</template>
```

Then import `axios` and the `ImageGallery` and set this up as a child component:

```javascript
<script>
import axios from 'axios';
import ImageGallery from '@/components/ImageGallery.vue';
export default {
  name: 'Home',
  components: {
    ImageGallery,
  },
}
</script>
```

Initialize the data we need:

```javascript
  data() {
    return {
      photos: [],
      error: '',
    }
  }
```

Add a `created()` hook and a `getPhotos()` method:

```javascript
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
  }
```

## Testing

You can now see the photos from all users on the home page.

![home page](/screenshots/home.png)

Kindly proceed to [Part 7](/tutorials/part7.md).
