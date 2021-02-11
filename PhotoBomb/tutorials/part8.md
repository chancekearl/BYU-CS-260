# Part 8: Comments

Modify the application so that users, if they are logged in, can comment on a
photo on its photo page. The requirements for this part are:

- The photo page should have a form for entering comments on the page.

- Comments are owned by a user. A user must be logged in to make comments on a
  photo. The form for adding comments is not displayed if the user is not logged
  in.

- Comments have a date.

- When comments are displayed, they show the user's name who made the comment
  and the date they made it. Format dates like we do for photos.

- When comments are added, they are displayed immediately on the page. You do
  not need to automatically display comments being added by a second user that
  is using the app at the same time (e.g. from a different computer). But you
  should be able to show the comments from the second user if you reload the
  page.

- The photo page use axios to add the comment and to
  fetch comments.

- The schema for comments will need to reference a User and a Photo.

- The server should have a new REST API for adding and getting comments for a
  photo. This API should use Mongoose to interact with the database. You will
  need to populate the database query for comments to fill in the user, like we
  did for photos.

## Hints

Recall that we setup a photo schema:

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
```

This references the `User` model. The comments schema should reference both a user and a photo.

When we retrieved a list of photos from Mongo, we used something like this:

```javascript
let photos = await Photo.find().sort({
  created: -1
}).populate('user');
```

This gets a list of all the photos in the database, sorts them, and then populates them with user records. This means that
the database finds the user id that is listed in the photo document, looks up the user document for that user in the database,
and then inserts it with the photo results. This way, if you have a photo `p1`, you can call `p1.user.name` and get the details
of that user.

Likewise, when you retrieve a set of comments, you can populate the results with the users, so that you can know which user
made which comment.

A tricky part of the endpoints for creating and getting comments is that comments reference both the user who created the comment and the photo they are commenting on.

* For the endpoint to create a comment, you can use the `validUser` middleware, and this will set req.user to the user record of the logged-in user. To know which photo they are commenting on, the endpoint can use `/:id` for the endpoint, and then req.params.id will be set to that photo ID. It is good practice to find the photo first that has the corresponding ID. After all, it might not exist! So be sure it exists first. Then, you can create the comment with:

```javascript
let comment = new Comment({
      comment: req.body.comment,
      user: req.user,
      photo: photo,
    });
```

where `photo` is a variable set to the photo you found using `Photo.findOne()`.

* For the endpoint to return a list of comments, you can use `/:id` for the endpoint again, and again make sure that photo exists in the database. If the photo exists, you can find all the comments with:

```javascript
let comments = await Comment.find({
  photo: photo
});
```

Remember that you can use `populate` to populate this with the corresponding user's record so the front end knows which user created each comment.
