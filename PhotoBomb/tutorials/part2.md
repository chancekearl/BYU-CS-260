# Part 2: Navigation

We will setup a navigation system using [Pure.css](https://purecss.io/).
The `Pure.css` styles provide a simple way to create a grid, forms, buttons,
and menus without having to load JavaScript. If you want a responsive menu
system, see their website for a very small pure JavaScript file you can include.

Start by running the front end:

```
npm run serve
```

## Cleanup

The first thing I like to do is remove much of the starter template. Do the
following:

* In `router/index.js`, remove the second route, the one for the About page.

* Remove `views/About.vue` and `components/HelloWorld.vue`.

* Modify `views/Home.vue` so it has just an empty div in the `template` and no
* references to the `HelloWorld` component in the `script` section.

## Site Title

Edit `public/index.html`. Add links for a Google Font, Pure CSS, and Font Awesome.
Also change the title of the site:

```html
  <link href="https://fonts.googleapis.com/css2?family=Work+Sans:wght@300;400&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="https://unpkg.com/purecss@1.0.1/build/pure-min.css" integrity="sha384-oAOxQR6DkCoMliIh8yFnu25d7Eq/PHS21PClpwjOTeU2jRSq11vu66rf90/cZr47" crossorigin="anonymous">
  <link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.13.0/css/all.css">
  <title>Photo Bomb</title>
```

*Note: any time you use a library, as above, check that you have the most
*up-to-date version.*

## Navigation

Now let's create a menu.

In `src/App.vue`, modify the `template` section so it looks like this:

```html
<template>
  <div id="app">
    <header id="header">
      <h1>Photo Bomb</h1>
      <nav>
        <router-link to="/"><i class="fas fa-home"></i></router-link>
        <router-link to="/dashboard"><i class="fas fa-user"></i></router-link>
      </nav>
    </header>
    <router-view />
  </div>
</template>
```

We're going to do something a little different with this menu. Add
the following `style` section:

```html
<style>
/* Color scheme: https://paletton.com/#uid=7040u0knHs+edG7jrvYscpiuCk2 */
/* red: #e74c3c
 * blue: #277E8E
 */
body {
  font-family: 'Work Sans', sans-serif;
  font-weight: 300;
  font-size: 13pt;
  margin: 0px 200px;
}

#header {
  /* Semi-circle */
  margin: 0 1em 1em 0;
  height: 100px;
  width: 200px;
  border-bottom-left-radius: 200px;
  border-bottom-right-radius: 200px;
  /* Fixed position */
  position: fixed;
  z-index: 10000;
  left: 50%;
  transform: translate(-50%, 0);
  /* Color and alignment */
  background: #e74c3c;
  text-align: center;
  box-shadow: 0 0 0 1em #FFF;
}

nav {
  display: flex;
  justify-content: center;
}

h1 {
  color: #fff;
  font-size: 18px;
}

h2 {
  font-size: 12px;
}

#header .fas {
  font-size: 25px;
  color: #fff;
  width: 50px;
}

.pure-button-primary {
  background-color: #277E8E;
}
</style>
```

You should see the menu for this site:

![menu](/screenshots/menu.png)

I've provided some comments in the CSS to show you how I made a half circle and placed it. The rest of the positioning is with flexbox and the menu icons are from Font Awesome.

Kindly proceed to [Part 3](/tutorials/part3.md).
