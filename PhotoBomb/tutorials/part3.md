# Part 3: Server Setup

We need to setup a server for the back end of this project.

In the top level of
this project, create a directory for the back end:

```
mkdir back-end
cd back-end
```

Now install some packages we will use:

```
npm install express mongoose argon2 body-parser cookie-parser cookie-session multer
```

Now create a file
called `server.js` with the following content:

```javascript
const express = require('express');
const bodyParser = require("body-parser");
const mongoose = require('mongoose');

// setup express
const app = express();

// setup body parser middleware to conver to JSON and handle URL encoded forms
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({
  extended: false
}));

// connect to the mongodb database
mongoose.connect('mongodb://localhost:27017/photobomb', {
  useUnifiedTopology: true,
  useNewUrlParser: true
});

const cookieParser = require("cookie-parser");
app.use(cookieParser());

const cookieSession = require('cookie-session');
app.use(cookieSession({
  name: 'session',
  keys: [
    'secretValue'
  ],
  cookie: {
    maxAge: 24 * 60 * 60 * 1000 // 24 hours
  }
}));

app.listen(3001, () => console.log('Server listening on port 3001!'));
```

This sets up Express, the body parser middleware, Mongo, the cookie parser, and cookie sessions. See previous
activities for explanations of these.

## Vue proxy server

Go back to the front end:

```
cd front-end
```

In this directory, create a file called `vue.config.js`, containing the following:

```javascript
module.exports = {
  devServer: {
    proxy: {
      '^/api': {
        target: 'http://localhost:3001',
      },
    }
  }
}
```

This lets the webpack development server that is started by `npm run serve` proxy the requests for the API and send them to your node server.

Note we are using a different port this time. This will ensure it does not
conflict with your previous projects.

Kindly proceed to [Part 4](/tutorials/part4.md).
