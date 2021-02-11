# Part 4: Authentication

Users need to register for an account and login.

We will use nearly the same code from our activity on [authenticating users](https://github.com/BYU-CS-260/authenticating-users). Please see that activity for explanations of this code.

The only change to this code is to remove roles from user accounts since we won't use them here.

## Back end

Create a file in the `back-end` directory called `users.js` and put the following there:

```javascript
const express = require("express");
const mongoose = require('mongoose');
const argon2 = require("argon2");

const router = express.Router();

//
// User schema and model
//

// This is the schema. Users have usernames and passwords. We solemnly promise to
// salt and hash the password!
const userSchema = new mongoose.Schema({
  firstName: String,
  lastName: String,
  username: String,
  password: String,
});

// This is a hook that will be called before a user record is saved,
// allowing us to be sure to salt and hash the password first.
userSchema.pre('save', async function(next) {
  // only hash the password if it has been modified (or is new)
  if (!this.isModified('password'))
    return next();

  try {
    // generate a hash. argon2 does the salting and hashing for us
    const hash = await argon2.hash(this.password);
    // override the plaintext password with the hashed one
    this.password = hash;
    next();
  } catch (error) {
    console.log(error);
    next(error);
  }
});

// This is a method that we can call on User objects to compare the hash of the
// password the browser sends with the has of the user's true password stored in
// the database.
userSchema.methods.comparePassword = async function(password) {
  try {
    // note that we supply the hash stored in the database (first argument) and
    // the plaintext password. argon2 will do the hashing and salting and
    // comparison for us.
    const isMatch = await argon2.verify(this.password, password);
    return isMatch;
  } catch (error) {
    return false;
  }
};

// This is a method that will be called automatically any time we convert a user
// object to JSON. It deletes the password hash from the object. This ensures
// that we never send password hashes over our API, to avoid giving away
// anything to an attacker.
userSchema.methods.toJSON = function() {
  var obj = this.toObject();
  delete obj.password;
  return obj;
}

// create a User model from the User schema
const User = mongoose.model('User', userSchema);

/* Middleware */

// middleware function to check for logged-in users
const validUser = async (req, res, next) => {
  if (!req.session.userID)
    return res.status(403).send({
      message: "not logged in"
    });
  try {
    const user = await User.findOne({
      _id: req.session.userID
    });
    if (!user) {
      return res.status(403).send({
        message: "not logged in"
      });
    }
    // set the user field in the request
    req.user = user;
  } catch (error) {
    // Return an error if user does not exist.
    return res.status(403).send({
      message: "not logged in"
    });
  }

  // if everything succeeds, move to the next middleware
  next();
};

/* API Endpoints */

/* All of these endpoints start with "/" here, but will be configured by the
   module that imports this one to use a complete path, such as "/api/user" */

// create a new user
router.post('/', async (req, res) => {
  // Make sure that the form coming from the browser includes all required fields,
  // otherwise return an error. A 400 error means the request was
  // malformed.
  if (!req.body.firstName || !req.body.lastName || !req.body.username || !req.body.password)
    return res.status(400).send({
      message: "first name, last name, username and password are required"
    });

  try {

    //  Check to see if username already exists and if not send a 403 error. A 403
    // error means permission denied.
    const existingUser = await User.findOne({
      username: req.body.username
    });
    if (existingUser)
      return res.status(403).send({
        message: "username already exists"
      });

    // create a new user and save it to the database
    const user = new User({
      firstName: req.body.firstName,
      lastName: req.body.lastName,
      username: req.body.username,
      password: req.body.password
    });
    await user.save();
    // set user session info
    req.session.userID = user._id;

    // send back a 200 OK response, along with the user that was created
    return res.send({
      user: user
    });
  } catch (error) {
    console.log(error);
    return res.sendStatus(500);
  }
});

// login a user
router.post('/login', async (req, res) => {
  // Make sure that the form coming from the browser includes a username and a
  // password, otherwise return an error.
  if (!req.body.username || !req.body.password)
    return res.sendStatus(400);

  try {
    //  lookup user record
    const user = await User.findOne({
      username: req.body.username
    });
    // Return an error if user does not exist.
    if (!user)
      return res.status(403).send({
        message: "username or password is wrong"
      });

    // Return the SAME error if the password is wrong. This ensure we don't
    // leak any information about which users exist.
    if (!await user.comparePassword(req.body.password))
      return res.status(403).send({
        message: "username or password is wrong"
      });

    // set user session info
    req.session.userID = user._id;

    return res.send({
      user: user
    });

  } catch (error) {
    console.log(error);
    return res.sendStatus(500);
  }
});

// get logged in user
router.get('/', validUser, async (req, res) => {
  try {
    res.send({
      user: req.user
    });
  } catch (error) {
    console.log(error);
    return res.sendStatus(500);
  }
});

// logout
router.delete("/", validUser, async (req, res) => {
  try {
    req.session = null;
    res.sendStatus(200);
  } catch (error) {
    console.log(error);
    return res.sendStatus(500);
  }
});


module.exports = {
  routes: router,
  model: User,
  valid: validUser
};
```

In addition, modify `server/server.js` so it contains the following, before
`app.listen`.

```javascript
// import the users module and setup its API path
const users = require("./users.js");
app.use("/api/users", users.routes);
```

## Front end

We'll create a front end that provides a way for users to register and login.

We're going to need `axios`, so install that now:

```
cd front-end
npm install axios
```

### User state

To keep track of the currently logged in user, modify `main.js` so it has the following:

```javascript
let data = {
  user: null
}

new Vue({
  data,
  router,
  render: h => h(App)
}).$mount('#app')
```

### Dashboard

Start by creating a view in `views/Dashboard.vue`. We will eventually use this to show a user's photo stream when they are logged in. But when not logged in, it should show registration and login forms.

Place the following in the `template` section of this new view:

```html
<template>
<div class="dashboard">
  <MyPhotos v-if="user" />
  <Login v-else />
</div>
</template>
```

This will load a `MyPhotos` component to show the photo stream of the logged in user, or a `Login` component if they are not logged in.

Place this in the `script` section of this component:

```javascript
<script>
import MyPhotos from '@/components/MyPhotos.vue';
import Login from '@/components/Login.vue';
import axios from 'axios';
export default {
  name: 'dashboard',
  components: {
    MyPhotos,
    Login,
  },
  async created() {
    try {
      let response = await axios.get('/api/users');
      this.$root.$data.user = response.data.user;
    } catch (error) {
      this.$root.$data.user = null;
    }
  },
  computed: {
    user() {
      return this.$root.$data.user;
    }
  }
}
</script>
```

This looks a lot like the home page from the [authenticating users](https://github.com/BYU-CS-260/authenticating-users) activity. We use a `created()` hook to get the user record if the browser has a cookie keeping a previous login active. The user's record is stored in global data. Then, the dashboard shows either the `MyPhotos` component (if logged in), or the `Login` component.

Add this style as well:

```html
<style scoped>
.dashboard {
  padding-top: 10px;
}
</style>
```

### Registration and login

Create a new component in `components/Login.vue` and add the following template:

```javascript
<template>
<div class="hero">
  <div class="heroBox">
    <form class="pure-form">
      <fieldset>
        <legend>Register for an account</legend>
        <input placeholder="first name" v-model="firstName">
        <input placeholder="last name" v-model="lastName">
      </fieldset>
      <fieldset>
        <input placeholder="username" v-model="username">
        <input type="password" placeholder="password" v-model="password">
      </fieldset>
      <fieldset>
        <button type="submit" class="pure-button pure-button-primary" @click.prevent="register">Register</button>
      </fieldset>
    </form>
    <p v-if="error" class="error">{{error}}</p>
    <form class="pure-form space-above">
      <fieldset>
        <legend>Login</legend>
        <input placeholder="username" v-model="usernameLogin">
        <input type="password" placeholder="password" v-model="passwordLogin">
      </fieldset>
      <fieldset>
        <button type="submit" class="pure-button pure-button-primary" @click.prevent="login">Login</button>
      </fieldset>
    </form>
    <p v-if="errorLogin" class="error">{{errorLogin}}</p>
  </div>
</div>
</template>
```

This creates two forms, one for registration and another for login, along with associated error messages and event handlers.

Here is the `script` section, a piece at a time:

```javascript
<script>
import axios from 'axios';
export default {
  name: 'HomePage',
  data() {
    return {
      firstName: '',
      lastName: '',
      username: '',
      password: '',
      usernameLogin: '',
      passwordLogin: '',
      error: '',
      errorLogin: '',
    }
  },
}
</script>
```

This imports `axios` so we can use it for requests we send to the back end. It also initializes all the data properties this view will use.

Next:

```javascript
  methods: {
    async register() {
      this.error = '';
      this.errorLogin = '';
      if (!this.firstName || !this.lastName || !this.username || !this.password)
        return;
      try {
        let response = await axios.post('/api/users', {
          firstName: this.firstName,
          lastName: this.lastName,
          username: this.username,
          password: this.password,
        });
        this.$root.$data.user = response.data.user;
      } catch (error) {
        this.error = error.response.data.message;
        this.$root.$data.user = null;
      }
    },
  }
```

This defines a `register()` method, which collects the information from the registration form and sends it to the back end to create a new user with the `POST /api/users` endpoint. If this is successful, the returned user record is stored in global data. Any errors are shown in the template.

Finally:

```javascript
    async login() {
      this.error = '';
      this.errorLogin = '';
      if (!this.usernameLogin || !this.passwordLogin)
        return;
      try {
        let response = await axios.post('/api/users/login', {
          username: this.usernameLogin,
          password: this.passwordLogin,
        });
        this.$root.$data.user = response.data.user;
      } catch (error) {
        this.errorLogin = "Error: " + error.response.data.message;
        this.$root.$data.user = null;
      }
    },
```

This defines a `login()` method,  which collects the information from the login form and sends it to the back end to create a new user with the `POST /api/users/login` endpoint. If this is successful, the returned user record is stored in global data. Any errors are shown in the template.

Following are the styles for this component:

```html
<style scoped>
.space-above {
  margin-top: 50px;
}

h1 {
  font-size: 28px;
  font-variant: capitalize;
}

.hero {
  padding: 120px;
  display: flex;
  justify-content: center;
}

.heroBox {
  text-align: center;
}

.hero form {
  font-size: 14px;
}

.hero form legend {
  font-size: 20px;
}

input {
  margin-right: 10px;
}

.error {
  margin-top: 10px;
  display: inline;
  padding: 5px 20px;
  border-radius: 30px;
  font-size: 10px;
  background-color: #d9534f;
  color: #fff;
}
</style>
```

### Logout

We now need to add the ability to show the user they are logged in and allow them to logout.

Create a new component in `components/MyPhotos.vue` and place the following template there:

```html
<template>
<div class="main">
  <div class="menu">
    <p><a><i class="fas fa-image"></i></a></p>
    <h2>{{user.firstName}} {{user.lastName}} <a @click="logout"><i class="fas fa-sign-out-alt"></i></a></h2>
  </div>
</div>
</template>
```

This shows a button for uploading photos (to be completed later) and a button for logging out.

Use the following for the `script` portion:

```javascript
<script>
import axios from 'axios';
export default {
  name: 'MyPhotos',
  data() {
    return {}
  },
  computed: {
    user() {
      return this.$root.$data.user;
    },
  },
  methods: {
    async logout() {
      try {
        await axios.delete("/api/users");
        this.$root.$data.user = null;
      } catch (error) {
        this.$root.$data.user = null;
      }
    },
  }
}
</script>
```

A computed property provides access to the current user's record. The `logout` method uses the `DELETE /api/users` endpoint to logout the user.

Add some style for this component:

```html
<style scoped>
.menu {
  display: flex;
  justify-content: space-between;
}

.menu h2 {
  font-size: 14px;
}
</style>
```

### Routing

We need to setup a route for this component so that the `/dashboard` URL will go to the `Dashboard` view. In `router/index.js`, add the import statement for this view:

```javascript
import Dashboard from '../views/Dashboard.vue'
```

Then add a configuration for the route:

```javascript
  {
    path: '/dashboard',
    name: 'Dashboard',
    component: Dashboard
  }
```

## Testing

If you click on the person icon in the menu, this should navigate to `/dashboard`. Since you don't have any accounts created yet, you should see the registration and login forms:

![registration and login](/screenshots/registration-and-login.png)

Open the browser console and select the network tab. When you register a new user, you should see the request to the API, with a `200 OK` status and a user record returned. In addition, the page should show that user now logged in:

![registration](/screenshots/registration.png)

Likewise, you can use the logout button and then login with this new user, and see the corresponding requests to the API for this:

![login](/screenshots/login.png)

If you do not see this functionality working, then you can debug by examining the network requests, finding the errors, and then hunting down any errors logged on the browser console or the terminal where the back end is running.

Kindly proceed to [Part 5](/tutorials/part5.md).
