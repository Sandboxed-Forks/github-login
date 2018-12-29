# How to do 3-legged OAuth with GitHub, a general guide by example with Node.js

![](https://i1.wp.com/shiya.io/wp-content/uploads/2018/02/github-login.gif?w=660)

Try the [live demo](https://login-with-github.herokuapp.com/) of the working tutorial.

3-legged authorization(OAuth) is the “Login with XXX” button you see everywhere. It’s a login protocol that lets the authentication provider authorize your application with the permission of the user. It redirects users to a new page, signs in, and approves the use of the user’s data. Some companies provide SDKs to make the process easier like Google and Facebook login, but if you learn the general process once, you can implement this for any API that requires 3-legged authentication.

Instead of implementing OAuth yourself, there are lots of frameworks out there like [Passport.js](http://www.passportjs.org/) or paid services like [Auth0](https://auth0.com/) for you to use in a production application. This post is intended to be a tutorial so you can learn it and not be too frustrated with the process.

Since the most popular APIs like Google and Facebook already provide custom SDKs, I’ll use GitHub’s OAuth API because it doesn’t have an officially recommended SDK. Their API doc page is very succinct, so if you’re already a seasoned Web developer, just head over to [their site](https://developer.github.com/apps/building-oauth-apps/authorization-options-for-oauth-apps/) and read their guide instead.  

## The process (no code yet)

1.  When the user wants to sign in, they click a button or link that says “Log in with GitHub.”
    
    In your app, you send a GET request to the `/login/oauth/authorize` endpoint, along with some query parameters: your application’s `client_id`, the `redirect_uri` (more on this later) and `scope` (permissions of the things your app would like to gain access to). These parameters are standard in almost all 3-legged OAuth implementations. GitHub also accepts two other parameters, the `state` string, to protect against [CSRF attacks](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)), and `allow_signup`, which lets you determine whether the sign-up link is present in the redirect.
    
    ![](https://i0.wp.com/shiya.io/wp-content/uploads/2018/02/login-click.gif?w=660)
    
2.  The user clicks the button, the page redirects and lets them enter their GitHub username/password and log in.
    
    Your application has no control over this interaction, hence the “Login with GitHub.” The idea is that as an application developer, you don’t want to worry about authentication yourself, like how and where to store user passwords. They will all be handled by GitHub.
    
    ![](https://i0.wp.com/shiya.io/wp-content/uploads/2018/02/login.gif?w=660)
    
3.  If this is the first time the user is granting access to your app, GitHub will confirm with them about the `scope`, or permissions to grant, which you specified in your previous API call.
    
    ![](https://i2.wp.com/shiya.io/wp-content/uploads/2018/02/authorize.png?w=660)
    
4.  Once access is granted and the user is logged in, GitHub will then send a request to your application to your callback URI.
    
    You can provide this URI in your GET request or specify it when you configure your app with GitHub. With it is a `code` parameter that identifies the login attempt, which is an identifier you send back later. Your application needs to be able to handle this request.
    
5.  With the `code` string, your app sends your `client_id` and `client_secret` to GitHub, and finally your app gets an access token from the response, which allows your app to access the user’s data from GitHub.
    

Let’s jump into the code.

## 1\. Configuring your app on GitHub

First of all, you need a developer account with GitHub. Go register at [developer.github.com](https://developer.github.com/), and then  
[register for a new OAuth app](https://github.com/settings/applications/new).

![](https://i2.wp.com/shiya.io/wp-content/uploads/2018/02/Screenshot-2018-01-29-16.49.33.png?w=660)

## 2\. Setting up your project

Install [Node](https://nodejs.org/en/) and [yarn](https://yarnpkg.com/en/) if you haven’t already. If you don’t have `yarn`, `npm` will do just fine – you can swap out all the `yarn` commands with `npm`.

In a new project folder, in your terminal/command line, type:

```
touch .gitignore app.js .env index.html views/styles.css
yarn init # or npm init, set up the package manager

```

Go to copy & paste the content of this [recommended gitignore file](https://github.com/github/gitignore/blob/master/Node.gitignore) into your `.gitignore`.

Copy and paste the client id and client secret you received when configuring your GitHub app in step 1, and paste it in `.env`:

```
CLIENT_ID=YOUR_CLIENT_ID
CLIENT_SECRET=YOUR_CLIENT_SECRET
HOST=http://localhost:3000 # makes it easier to deploy

```

## 3\. Installing dependencies

There are a few packages to install here:

```
yarn add dotenv express request randomstring express-session

```

*   `dotenv` makes handling of your config file easier. ([documentation](https://github.com/motdotla/dotenv))
*   `express` makes route handling easier. ([documentation](http://expressjs.com/))
*   `request` makes sending requests easier. ([documentation](https://github.com/request/request))
*   `randomstring` makes generating random strings easier. ([documentation](https://www.npmjs.com/package/randomstring))
*   `express-session` stores data into session so that different users can log in to your Node app in their own session. ([documentation](https://github.com/expressjs/session))

## 4\. The HTML and CSS

`index.html` is the page express sends you to when you visit your app at `localhost:3000`.

In `index.html`:

```
<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8" />
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <title>Log In to GitHub</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" type="text/css" media="screen" href="styles.css" />
</head>

<body>
  <a id="login-button" href="/login">Log In With GitHub</a>
</body>

</html>

```

This is how I styled my button but do w/e you want.

In `views/styles.css`:

```
#login-button {
  background-color: #3c4146;
  color: #EEF4EC;
  padding: 1em;
  position: fixed;
  top: 50%;
  left: 50%;
  margin-left: -100px;
  margin-top: -1em;
  text-decoration: none;
  font-family: Arial, Helvetica, sans-serif;
  width: 150px;
  text-align: center;
}

```

## 5\. The server code

Now let’s write some Node code in `app.js`.

You first need to require all the packages we installed earlier, and some Node core modules, and initialize them.

In `app.js`:

```
// save environment variables in dotenv
require('dotenv').config();

// express set up, handles request, response easily
const express = require('express');
const app = express();

// express session
const session = require('express-session');

// makes sending requests easy
const request = require('request');

// node core module, construct query string
const qs = require('querystring');

// node core module, parses url string into components
const url = require('url');

// generates a random string for the
const randomString = require('randomstring');

// random string, will be used in the workflow later
const csrfString = randomString.generate();

// setting up port and redirect url from process.env
// makes it easier to deploy later
const port = process.env.PORT || 3000;
const redirect_uri = process.env.HOST + '/redirect';

// serves up the contnests of the /views folder as static 
app.use(express.static('views'));

// initializes session
app.use(
  session({
    secret: randomString.generate(),
    cookie: { maxAge: 60000 },
    resave: false,
    saveUninitialized: false
  })
);

```

Create a route that serves the `index.html` file we created earlier when the user visits the root path, “`/`“.

In `app.js`:

```
app.get('/', (req, res, next) => {
  res.sendFile(__dirname + '/index.html');
});

app.listen(port, () => {
  console.log('Server listening at port ' + port);
});

```

Now if you go to `localhost:3000` with your browser, we should see the login button. It won’t work yet.

## 6\. Handling `/login`

If you noticed, in `index.html` the login button has an attribute `href="/login"`. If you click on it, the browser will try to take us to `localhost:3000/login`, which will throw an error.

Clicking on that button should redirect the user to GitHub and log in so they can authorize your application.

In `app.js`:

```
app.get('/login', (req, res, next) => {
    // generate that csrf_string for your "state" parameter
  req.session.csrf_string = randomString.generate();
    // construct the GitHub URL you redirect your user to.
    // qs.stringify is a method that creates foo=bar&bar=baz
    // type of string for you.
  const githubAuthUrl =
    'https://github.com/login/oauth/authorize?' +
    qs.stringify({
      client_id: process.env.CLIENT_ID,
      redirect_uri: redirect_uri,
      state: req.session.csrf_string,
      scope: 'user:email'
    });
  // redirect user with express
  res.redirect(githubAuthUrl);
});

```

Now if a user clicks on the button, it will direct them to GitHub, which will prompt them to log in and authorize the app. GitHub will then try to call your redirect URL. Nothing will happen, because your app can’t handle requests to that URL yet.

## 7\. Handling the redirect URL

If you had set up the redirect URL or authorization callback URL in your app configuration (in this case `http://localhost:3000/redirect`), GitHub will call it. You need to be able to handle these requests in your app.

GitHub will send your app a `code` parameter which you then use to call their `/login/oauth/access_token` API, along with your `client_secret` to get an access token.

After getting the access token, we will take the user to another page where we confirm that they authorized.

```
// Handle the response your application gets.
// Using app.all make sures no matter the provider sent you
// get or post request, they will all be handled
app.all('/redirect', (req, res) => {
  // Here, the req is request object sent by GitHub
  console.log('Request sent by GitHub: ');
  console.log(req.query);

  // req.query should look like this:
  // {
  //   code: '3502d45d9fed81286eba',
  //   state: 'RCr5KXq8GwDyVILFA6Dk7j0LbFNTzJHs'
  // }
  const code = req.query.code;
  const returnedState = req.query.state;

  if (req.session.csrf_string === returnedState) {
    // Remember from step 5 that we initialized
    // If state matches up, send request to get access token
    // the request module is used here to send the post request
    request.post(
      {
        url:
          'https://github.com/login/oauth/access_token?' +
          qs.stringify({
            client_id: process.env.CLIENT_ID,
            client_secret: process.env.CLIENT_SECRET,
            code: code,
            redirect_uri: redirect_uri,
            state: req.session.csrf_string
          })
      },
      (error, response, body) => {
        // The response will contain your new access token
        // this is where you store the token somewhere safe
        // for this example we're just storing it in session
        console.log('Your Access Token: ');
        console.log(qs.parse(body));
        req.session.access_token = qs.parse(body).access_token;

        // Redirects user to /user page so we can use
        // the token to get some data.
        res.redirect('/user');
      }
    );
  } else {
    // if state doesn't match up, something is wrong
    // just redirect to homepage
    res.redirect('/');
  }
});

```

Now we can retrieve that user’s email through GitHub’s API using this new access token! Remember that in step 6, we specified a `scope` parameter with the value `user: email`. That specifies the data that you now have access to.

In the previous code snippet, we redirected to `/user`. Let’s retrieve the user’s email from GitHub and show it on this page.

```
app.get('/user', (req, res) => {
  // GET request to get emails
  // this time the token is in header instead of a query string
  request.get(
    {
      url: 'https://api.github.com/user/public_emails',
      headers: {
        Authorization: 'token ' + req.session.access_token,
        'User-Agent': 'Login-App'
      }
    },
    (error, response, body) => {
      res.send(
        "<p>You're logged in! Here's all your emails on GitHub: </p>" +
        body +
        '<p>Go back to <a href="/">log in page</a>.</p>'
      );
    }
  );
});

```

Now after you log in, the app should greet you with the data fetched from GitHub.

The working code is [on GitHub](https://github.com/shiya/github-login).

## Gotchas

There are a lot of quirks when implementing OAuth so expect some roadbumps along the way. The good news is that it’s equally annoying for companies providing OAuth, so it’s unlikely they’ll deprecate or change the API. You won’t need to change it very often once it’s working 🙂

Here are some issues you might run into:

1.  Every API provider develops 3-legged authentication slightly differently. Sometimes the accepted parameters are in the header, sometimes in the query string.
    
2.  There’s also no agreement on the response format. Some companies provide a JSON response `{ foo: bar }`. Other companies, like GitHub, offer a query string response `foo=bar&bar=baz`.
    
3.  Be aware of API throttling and access token expiry time. These are two things I’ve seen API providers change on a whim. A good practice is to store the token somewhere safe and handle authentication errors by regenerating the token.
    
4.  Terminology can be inconsistent. Redirect URL and callback URL usually mean the same thing, and likewise `client_id`/`client_secret` and `app_id`/`app_secret`. There will always be a callback URL for the authentication provider to redirect to, and your app will always have a client id and secret.
    

Now you should be able to implement OAuth in your application! There’s a lot more that can be done, like writing integration tests and persisting your data in a database, but they aren’t in the scope of this tutorial.

Happy coding!

### Share this:

*   [Click to share on Twitter (Opens in new window)](http://shiya.io/how-to-do-3-legged-oauth-with-github-a-general-guide-by-example-with-node-js/?share=twitter "Click to share on Twitter")
*   [Click to share on LinkedIn (Opens in new window)](http://shiya.io/how-to-do-3-legged-oauth-with-github-a-general-guide-by-example-with-node-js/?share=linkedin "Click to share on LinkedIn")

### *Related*

Posted on [February 10, 2018February 10, 2018](http://shiya.io/how-to-do-3-legged-oauth-with-github-a-general-guide-by-example-with-node-js/)Author [Shiya Luo](http://shiya.io/author/shiya/)Categories [Node.js](http://shiya.io/category/node-js/), [Tutorial](http://shiya.io/category/tutorial/)

### Leave a Reply [Cancel reply](/how-to-do-3-legged-oauth-with-github-a-general-guide-by-example-with-node-js/#respond)

Your email address will not be published. Required fields are marked \*

Comment

Name \* 

Email \* 

Website 

 Notify me of follow-up comments by email.

 Notify me of new posts by email.

  

This site uses Akismet to reduce spam. [Learn how your comment data is processed](https://akismet.com/privacy/).

## Post navigation
