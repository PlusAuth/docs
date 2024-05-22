---
title: Financial NodeJS Application
description: This tutorial demonstrates how to create a financial NodeJS app using Express.
type: fintech
topics:
  - quickstarts
  - financial
  - fintech
  - nodejs
  - express
icon: i-devicon-express
label: Express
---

In this tutorial, you will see how to create a simple financial service in NodeJs using the Express framework.


## Configure PlusAuth
First of all, you need to create a Fintech Service client from the [PlusAuth Dashboard](https://dashboard.plusauth.com/~clients).
Unlike other application types, you need to define your JWKS (JSON Web Key Set) while creating a financial client for validating
client assertion. You will learn more about this later in this article.
If you would like to learn about the internals, you can look at [OpenID Connect Client Authentication](https://openid.net/specs/openid-connect-core-1_0-15.html#ClientAuthentication) section.


### Generating a JWK set
There are many tools that will help you to generate JWKS. We will be using [jose](https://www.npmjs.com/package/jose) for this tutorial.
Here is a simple node script that will generate a jwks and write them to corresponding files. Don't forget to install `jose` library before using the following script. 

```js
// generate_jwks.js

const fs = require('fs')
const jose = require('jose')

const key = jose.JWK.generateSync("EC", 'P-256', { use: "sig", alg: "ES256"})

const privateKey = key.toJWK(true)
const publicKey = key.toJWK(false)

fs.writeFileSync('es256_private.json', JSON.stringify(privateKey, null, 2));
fs.writeFileSync('es256_public.json', JSON.stringify(publicKey, null, 2));
```

After you run the above script, you will have two files which are our public and private keys. Make sure to keep your private key in a safe place.
Let's copy the public key to JWKS field in the PlusAuth dashboard client creation popup and finish the creation of the client.

### Configure Client
In order to continue this tutorial, you need to configure the redirect and post-logout redirect URIs of the client. You can also do this later.

Let's assume our application will be run on `localhost:3000`, and we will have `/auth/callback` route for OAuth2 redirect callback URI.
Go to client details from the dashboard and add `http://localhost:3000/auth/callback` to **Redirect Uri's**.

For logout redirect URI, lets use `/auth/logout/callback` endpoint. Your post-logout redirect URI will be `http://localhost:3000/auth/logout/callback`.
Let's add it to **Post Logout Redirect Uris** of the client and save the form.

## Create Node.js Application
We will be using Express web framework with pug templating engine. For environment-specific configuration, we will be using `dotenv` library.

### Install dependencies
So, here are the dependencies that we will be using.
- [express](https://www.npmjs.com/package/express) : NodeJS web framework
- [express-session](https://www.npmjs.com/package/express-session) : Session middleware for express apps
- [pug](https://www.npmjs.com/package/pug) : A templating engine
- [dotenv](https://www.npmjs.com/package/dotenv) : Environment variable loader utility
- [openid-client](https://www.npmjs.com/package/openid-client) : OpenID Connect client library with FAPI support
- [passport](https://www.npmjs.com/package/passport) : Authentication middleware for NodeJS

```shell
# install with npm
npm install express express-session pug dotenv openid-client passport
```
or with yarn
```shell
# install with yarn
yarn add express express-session pug dotenv openid-client passport
```

### Create `.env` file
Don't forget to replace values defined in the format of `<PLACEHOLDER>` according to your needs.

```properties
PLUSAUTH_ISSUER=https://<YOUR_TENANT>.plusauth.com
PLUSAUTH_CLIENT_ID=<YOUR_CLIENT_ID>
```

### Add `express-session` middleware
In our app, we will be using cookies to store user-session information.

```js
// app.js 

// Load environment variables
const dotenv = require("dotenv");
dotenv.config();

const session = require("express-session")

const sessionOptions = {
        cookie: {},
        resave: false,
        saveUninitialized: true,
        secret: "SomeRandomValue"
}

if(process.env.NODE_ENV === "production"){
    // Use secure cookies in production. For more: https://www.npmjs.com/package/express-session#cookiesecure
    sessionOptions.cookie.secure = true;
}

app.use(session(sessionOptions));
```

### Initialize FAPI client
We will create an OIDC client with FAPI support and will be using it in authorization middlewares. If you remember, we generated JWKS
in the step [Generating a JWK set](#generating-a-jwk-set). We will use generated private key here. So make sure you load it correctly.

```js
// app.js 

const fs = require("fs");
const { Issuer } = require("openid-client");

// Make sure es256_private.json file exists with your generated private key.

const privateKey = JSON.parse(fs.readFileSync("es256_private.json", { encoding: "utf-8" }))

const plusAuthIssuer = await Issuer.discover(process.env.AUTH_URL);

const plusAuthClient = new plusAuthIssuer.FAPIClient({
    client_id: process.env.CLIENT_ID,
    redirect_uris: ["http://localhost:3000/auth/callback"],
    post_logout_redirect_uris: ["http://localhost:3000/auth/logout/callback"],
    response_mode: "jwt",
    authorization_signed_response_alg: "PS256",
    id_token_signed_response_alg: "PS256",
    token_endpoint_auth_method: "private_key_jwt",
    request_object_signing_alg:"ES256",
}, { keys: [ privateKey ] } );
```

### Add passport authentication middleware
Let's include `passport` with the OIDC client's strategy and configure it accordingly to our needs.

```js
//app.js

const passport = require("passport");
const { Strategy, generators  } = require("openid-client");

const PlusAuthStrategy = new Strategy({
        client: plusAuthClient,
        params: {
            response_type: "code",
            response_mode: "jwt"
        },
        passReqToCallback: true
    },
    function(req, token, user, done) {
       // Store token to user session
       req.session.token = token
       return done(null, user)
    }
)

passport.use("PlusAuth", PlusAuthStrategy);

passport.serializeUser((user, next) => {
    next(null, user);
});

passport.deserializeUser((user, next) => {
    next(null, user);
});

app.use(passport.initialize())
app.use(passport.session())
```

## Create routes for login, logout, and user profile

### Login
We will be using the request object to pass authorization options in a FAPI conformant way to the PlusAuth.
```js
//app.js 

app.use("/auth/login", async function (req, res){
    const state = generators.state()
    const nonce = generators.nonce()
    // epoch time for 5 minutes later. It defines expiration of the request object.
    const in5minutes = new Date(new Date().getTime() + (5 * 60 * 1000) ).getTime() / 1000

    return passport.authenticate("PlusAuth", {
      state,
      nonce,
      request: await plusAuthClient.requestObject({
        state,
        nonce,
        exp: in5minutes,
        aud: process.env.PLUSAUTH_ISSUER,
        client_id: process.env.PLUSAUTH_CLIENT_ID,
        scope: "openid profile email",
        response_type: "code",
        redirect_uri: "http://localhost:3000/auth/callback",
      }),
    })(req,res)
});

app.use(
  "/auth/callback",
  (req,res,next) =>
      passport.authenticate("PlusAuth", {
        failureMessage: true,
        failureRedirect: "/error",
        successRedirect: "/profile",
      })(req, res, next)
);
```

### Logout
```js
app.get("/auth/logout", (req, res) => {
    res.redirect(
        plusAuthClient.endSessionUrl({ id_token_hint: req.user.id_token })
    );
});
app.get("/auth/logout/callback", (req, res) => {
    req.logout();
    res.redirect("/");
});
```

### Check auth middleware
```js
function isLoggedIn(req, res, next) {
    if (req.isAuthenticated()) {
      return next();
    }
    
    res.redirect("/auth/login");
}
```

### User Profile
```js
app.use("/profile", isLoggedIn, (req, res) => {
    res.render("profile", { user: req.user });
});
```

## Create views
We will be using [pug](https://pugjs.org/) as a template engine.

```js
// app.js
const path = require("path");

app.set("views", path.join(__dirname, "views"));
app.set("view engine", "pug");
```

Here is a simple layout with bootstrap.
```pug
// views/layout.pug
doctype html
html(lang='en')
    head
        meta(charset='utf-8')
        meta(name='viewport' content='width=device-width, initial-scale=1, shrink-to-fit=no')
        title Plusauth Starter Template
        link(rel='stylesheet' href='https://stackpath.bootstrapcdn.com/bootstrap/4.5.0/css/bootstrap.min.css' integrity='sha384-9aIt2nRpC12Uk9gS9baDl411NQApFmC26EwAOH8WgZl5MYYxFfc+NcPb1dKGj7Sk' crossorigin='anonymous')
    body
        style.
            body {
                padding-top: 5rem;
            }
        nav.navbar.navbar-expand-md.navbar-dark.bg-dark.fixed-top
            a.navbar-brand(href='/') Plusauth Starter
            #navbarsExampleDefault.collapse.navbar-collapse
                ul.navbar-nav.mr-auto
                if locals.user
                    li.nav-item.navbar-nav
                        a.nav-link(href='/profile')
                            | Logged in as: #{user.email}
                    a.nav-link(href='/auth/logout')
                        | Logout
                else
                    li.nav-item.navbar-nav
                        a.nav-link(href='/auth/login')
                            | Login
        main.container(role='main')
            block main
```

And following is our main page.
```pug
// views/index.pug
extends layout
block main
    .jumbotron
        .container
            h1.display-3 Hello, world!
            p
                | This is a simple Express application to demonstrate
                | how to use financial application in PlusAuth
            p
                if locals.user
                    a.btn.btn-success.btn-lg(href='/profile' role='button') View Profile »
                else
                    a.btn.btn-primary.btn-lg(href='/auth/login' role='button') Login/Register »
            p

```

### Profile
On the profile page, we will be printing authenticated user information as JSON with a welcome message.

```pug
// views/profile.pug
extends layout

block main
    .container
        h3 Welcome #{locals.user.email}
        pre.
            User object: #{ JSON.stringify(locals.user, null, 2) }

```

## See the results
Now you are ready to run your application. After installing dependencies, run your application. If you have followed this
tutorial exactly the application should run at [http://localhost:3000](http://localhost:3000)

The source code is served at GitHub. You can reach it from [here](https://github.com/PlusAuth/plusauth-node-financial-example)
