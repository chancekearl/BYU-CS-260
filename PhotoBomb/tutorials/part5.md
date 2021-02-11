# Part 5: Uploading Photos

We are now ready to have users upload photos.

# Back End

On the back end, we're going to use the
[multer](https://github.com/expressjs/multer) library to upload images.

First, create a file in `back-end/photos.js` that has the following:

```javascript
const mongoose = require('mongoose');
const express = require("express");
const router = express.Router();

// Configure multer so that it will upload to '/public/images'
const multer = require('multer')
const upload = multer({
  dest: '../front-end/public/images/',
  limits: {
    fileSize: 10000000
  }
});
```

This imports the necessary libraries and configures multer. Images will be
stored in `../public/images`.

Next, import the User model:

```javascript
const users = require("./users.js");
const User = users.model;
const validUser = users.valid;
```

Then, define a Photo schema and model:

```javascript
const photoSchema = new mongoose.Schema({
  user: {
    type: mongoose.Schema.ObjectId,
    ref: 'User'
  },
  path: String,
  title: String,
  description: String,
  created: {
    type: Date,
    default: Date.now
  },
});

const Photo = mongoose.model('Photo', photoSchema);
```

Notice that the `Photo` schema uses a reference to a `User`. This is what we'll use to ensure that photos belong to a particular user. We can use this reference to look up all the photos belonging to a user or ensure that a user can only
delete their own photos.

Notice also that both the `user` and `created` fields in the document use an
extended syntax to include both the type and other fields -- a reference for
the user and a default for the date the photo was created.

Now we need to define the API. First, uploading a photo:

```javascript
// upload photo
router.post("/", validUser, upload.single('photo'), async (req, res) => {
  // check parameters
  if (!req.file)
    return res.status(400).send({
      message: "Must upload a file."
    });

  const photo = new Photo({
    user: req.user,
    path: "/images/" + req.file.filename,
    title: req.body.title,
    description: req.body.description,
  });
  try {
    await photo.save();
    return res.sendStatus(200);
  } catch (error) {
    console.log(error);
    return res.sendStatus(500);
  }
});
```

We user the `validUser` middleware to validate the user is logged in. Assuming they are, we let them upload a file by passing control the multer `upload` function. The request includes all of
the metadata for the photo, so we can create a photo document and save this to
the database. We don't need to set the date since it will default to the current
date.

Next, create a method to handle getting all the photos for a given user:

```javascript
// get my photos
router.get("/", validUser, async (req, res) => {
  // return photos
  try {
    let photos = await Photo.find({
      user: req.user
    }).sort({
      created: -1
    }).populate('user');
    return res.send(photos);
  } catch (error) {
    console.log(error);
    return res.sendStatus(500);
  }
});
```

We again validate the user, which ensures there is a `req.user` object. Then we find all the photos that belong to that user
by specifying to `find` that we want all photos with `{user: req.user}`. We sort
this list by the date they were created in descending order (most recent photo
first).

We also do something new with this query: we call `populate()` to fill in the corresponding user's record (the user that owns this photo). This will replace the user's ID with an object representing the user.

We return this list in the response.

Finally, export the model and the routes:

```javascript
module.exports = {
  model: Photo,
  routes: router,
}
```

The last step for the back end is to import the photos module in `server.js`:

```javascript
const photos = require("./photos.js");
app.use("/api/photos", photos.routes);
```

Be sure to put this *before* the call for `listen()`.

## Front end

Now we need to modify the `MyPhotos.vue` component to upload and display photos for a user.
Instead of putting everything there, we are going to use several child
components. A component is a reusable piece of the application.

### Uploader component

Create a file in `src/components/Uploader.vue` and add this to the `template`
section:

```html
<template>
<transition v-if="show" name="modal">
  <div class="modal-mask">
    <div class="modal-container">
      <form class="pure-form" @submit.prevent="upload">
        <legend>Upload a picture</legend>
        <fieldset>
          <input v-model="title" placeholder="Title">
        </fieldset>
        <fieldset>
          <textarea v-model="description" placeholder="Description"></textarea>
        </fieldset>
        <fieldset>
          <div class="imageInput" @click="chooseImage">
            <img v-if="url" :src="url" />
            <div v-if="!url" class="placeholder">
              Choose an Image
            </div>
            <input class="fileInput" ref="fileInput" type="file" @input="fileChanged">
          </div>
          <p v-if="error" class="error">{{error}}</p>
        </fieldset>
        <fieldset class="buttons">
          <button type="button" @click="close" class="pure-button">Close</button>
          <button type="submit" class="pure-button pure-button-primary right">Upload</button>
        </fieldset>
      </form>
    </div>
  </div>
</transition>
</template>
```

This uses a [Vue transition](https://vuejs.org/v2/guide/transitions.html) to pop up a modal dialog for uploading a photo. In Vue, a transition allows you to automatically apply [CSS classes for transitions](https://www.w3schools.com/css/css3_transitions.asp).

We use the `modal` classes shown above to create a modal dialog, which is a dialogue that overlays the entire screen and focuses the user attention. When they are done, it is dismissed and the screen is shown again.

We use a
regular form with Vue events to handle form submission and closing the form.

For the script section of this component, add the following:

```javascript
<script>
import axios from 'axios';
export default {
  name: 'Uploader',
  props: {
    show: Boolean,
  },
  data() {
    return {
      title: '',
      description: '',
      url: '',
      file: null,
      error: '',
    }
  }
}
</script>
```

We define `props` for the component, which allows a parent component to pass a child component some data.
We use props just like data properties or computed properties. Notice
that props are typed and Vue applies type checking to these.

Add the following methods to the `script` section:

```javascript
methods: {
  fileChanged(event) {
    this.file = event.target.files[0];
    this.url = URL.createObjectURL(this.file);
  },
  close() {
    this.$emit('close');
  },
  chooseImage() {
      this.$refs.fileInput.click()
    },
}
```

The `fileChanged` event keeps track of the currently selected file. It also uses a [URL object](https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL) to convert the file contents into a local URL that can be used for previewing the image on the screen. This way the user sees the file they are uploading before sending the upload to the back end.

A new thing we show here is that the close
method emits an event called `close`. This is how child components can trigger
methods on their parent components. This one tells the parent component to close the uploader modal dialog.

The `chooseImage()` function uses a click event on the entire `fileInput` div to trigger a click on the file input (which is hidden by CSS). This works by using a `ref` attribute on the file input, then accessing it with `this.$refs`. This is a clever trick to have the user click on a div instead of a button.

Add one more method to the `script` section:

```javascript
    async upload() {
      try {
        const formData = new FormData();
        formData.append('photo', this.file, this.file.name);
        formData.append('title', this.title);
        formData.append('description', this.description);
        await axios.post("/api/photos", formData);
        this.file = null;
        this.url = "";
        this.title = "";
        this.description = "";
        this.$emit('uploadFinished');
      } catch (error) {
        this.error = "Error: " + error.response.data.message;
      }
    }
```

This handles the file upload. We must use a `FormData` object to send the
file to the server. This is handled by multer and transformed into `req.file` on the back end. We can append additional metadata such as a title and
description. These are accessed as `req.body.title` and `req.body.description`
on the back end.

If no error occurs, we close and reset the form and emit an `uploadFinished`
event to the parent. This will let the parent know when the upload is complete.

Finally, add a `style` section to this component:

```html
<style scoped>
/* Modals */
.modal-mask {
  position: fixed;
  z-index: 9998;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, .3);
  transition: opacity 0.5s ease;
  display: flex;
  transition: background 0.2s ease-in-out, height 1s ease-in-out;
  transition: height 0.2s ease-in-out, width 0.2s ease-in-out;
  justify-content: center;
  transition-timing-function: cubic-bezier(0.64, 0.57, 0.67, 1.53);
}

.modal-container {
  width: 50%;
  height: max-content;
  margin-top: 80px;
  padding: 20px 30px;
  background-color: #fff;
  border-radius: 2px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, .33);
  transition: all 0.5s ease;
}

/*
* The following styles are auto-applied to elements with
* transition="modal" when their visibility is toggled
* by Vue.js.
*
* You can easily play with the modal transition by editing
* these styles.
*/
.modal-enter {
  opacity: 0;
}

.modal-leave-active {
  opacity: 0;
}

.modal-enter .modal-container,
.modal-leave-active .modal-container {
  -webkit-transform: scale(1.1);
  transform: scale(1.1);
}

/* Form */

form {
  font-size: 11pt;
}

input {
  width: 100%;
}

textarea {
  width: 100%;
  height: 100px;
}

.placeholder {
  background: #F0F0F0;
  width: 200px;
  height: 125px;
  display: flex;
  justify-content: center;
  align-items: center;
  color: #333;
  font-size: 14px;
  cursor: pointer;
}

.placeholder:hover {
  background: #E0E0E0;
}

.fileInput {
  display: none;
}

img {
  width: 200px;
}

.buttons {
  display: flex;
  justify-content: space-between;
}
</style>
```

This is rather long because styling the modal dialog is fairly complicated.

### Image gallery component

We also want to create an image gallery component that we can use to show the photos after they are uploaded. This is something we can reuse on the front page eventually.

Create a file in `components/ImageGallery.vue` that contains the following template:

```html
<template>
<div>
  <section class="image-gallery">
    <div class="image" v-for="photo in photos" v-bind:key="photo._id">
      <img :src="photo.path" />
      <div class="photoInfo">
        <p class="photoTitle">{{photo.title}}</p>
        <p class="photoName">{{photo.user.firstName}} {{photo.user.lastName}}</p>
      </div>
      <p class="photoDate">{{formatDate(photo.created)}}</p>
    </div>
  </section>
</div>
</template>
```

This shows a set of photos, wrapped in a section with an `image-gallery` class.

Next, install the `moment` library. Be sure to do this in the `front-end` directory:

```
cd front-end
npm install moment
```

Place this in the `script` section:

```javascript
<script>
import moment from 'moment';

export default {
  name: 'ImageGallery',
  props: {
    photos: Array
  },
  methods: {
    formatDate(date) {
      if (moment(date).diff(Date.now(), 'days') < 15)
        return moment(date).fromNow();
      else
        return moment(date).format('d MMMM YYYY');
    }
  }
}
</script>
```

This declares the `props` for the `ImageGallery` component, which is an Array of photos. We also define a method to format the date. If it is relatively recent, we show the date as within the past few minutes, hours, or days. Otherwise we format the date normally.

Use this for the `style` section:

```html
<style scoped>
.photoInfo {
  display: flex;
  justify-content: space-between;
  font-size: 0.8em;
}

.photoInfo p {
  margin: 0px;
}

.photoDate {
  font-size: 0.7em;
  font-weight: normal;
}

p {
  margin: 0px;
}

/* Masonry */
*,
*:before,
*:after {
  box-sizing: inherit;
}

.image-gallery {
  column-gap: 1em;
}

.image {
  margin: 0 0 1.5em;
  display: inline-block;
  width: 100%;
}

.image img {
  width: 100%;
}

/* Masonry on large screens */
@media only screen and (min-width: 1024px) {
  .image-gallery {
    column-count: 4;
  }
}

/* Masonry on medium-sized screens */
@media only screen and (max-width: 1023px) and (min-width: 768px) {
  .image-gallery {
    column-count: 3;
  }
}

/* Masonry on small screens */
@media only screen and (max-width: 767px) and (min-width: 540px) {
  .image-gallery {
    column-count: 2;
  }
}
</style>
```

This uses a Masonry layout for the photos.

### MyPhotos component

We need to use both the `Uploader` and the `ImageGallery` component in the `MyPhotos` component. Start by modifying the
header in `components/MyPhotos.vue`:

```html
<template>
<div class="main">
  <div class="menu">
    <p><a @click="toggleUpload"><i class="fas fa-image"></i></a></p>
    <h2>{{user.firstName}} {{user.lastName}} <a @click="logout"><i class="fas fa-sign-out-alt"></i></a></h2>
    <uploader :show="show" @close="close" @uploadFinished="uploadFinished" />
  </div>
  <image-gallery :photos="photos" />
  <p v-if="error">{{error}}</p>
</div>
</template>
```

We add a click method for the upload icon. This toggles a `show` property that causes the `Uploader` component to show itself.

To pass data to the
`Uploader`, the `MyPage` view binds `show` to a local variable. This gets
passed as the `show` property in the component. To setup events that the
child component -- `Uploader` -- can call on the parent view -- `MyPhotos`,
we bind several event handlers. These match the `emitted` events in the child component.

We also include the `ImageGallery` component, passing it an array of photos for its props..

Next, we setup the `script` component of `MyPhotos` to import the
components:

```javascript
import axios from 'axios';
import Uploader from '@/components/Uploader.vue';
import ImageGallery from '@/components/ImageGallery.vue';
```

We also configure the Uploader and ImageGallery in a list of child components:

```javascript
export default {
  name: 'MyPhotos',
  components: {
    Uploader,
    ImageGallery
  }
}
```

And we need to initialize a few properties:

```javascript
  data() {
    return {
      show: false,
      photos: [],
      error: '',
    }
  },
```

Notice that the component in JavaScript is called `Uploader` and in HTML it
is called `uploader`. Likewise, `ImageGallery` becomes `image-gallery`.

Add a `created()` hook to get an array of photos when this page is loaded:

```javascript
created() {
    this.getPhotos();
  },
```

Add a method to get the photos:

```javascript
    async getPhotos() {
      try {
        this.response = await axios.get("/api/photos");
        this.photos = this.response.data;
      } catch (error) {
        this.error = error.response.data.message;
      }
    },
```

Next, add several methods to toggle the uploader:

```javascript
    close() {
      this.show = false;
    },
    toggleUpload() {
      this.show = true;
    },
```

Finally, add a method to handle the event that occurs when the uploader finishes:

```javascript
    async uploadFinished() {
      this.show = false;
      this.getPhotos();
    },
```

## Testing

Users that are logged in can now upload and see their own photos.

![uploading a photo](/screenshots/upload.png)

![uploaded photos](/screenshots/uploaded-photos.png)

Kindly proceed to [Part 6](/tutorials/part6.md).
