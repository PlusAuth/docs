---
title: PHP Tutorial
description: This tutorial demonstrates how to add user login, logout, and profile to a PHP application.
topics:
- quickstarts
- webapp
- login
- user profile
- logout
- php
- php7
contentType: tutorial
useCase: quickstart
type: web
icon: i-devicon-php
label: PHP
---
This tutorial shows how to use PlusAuth with PHP and `Jumbojett\OpenIDConnectClient`. If you do not have a PlusAuth account, register from [here](https://dashboard.plusauth.com).

::alert{type=info}
This tutorial follows [plusauth-php-starter](https://github.com/PlusAuth/plusauth-php-starter) sample project on Github. You can download and follow the tutorial via the sample project.
::

## Create PlusAuth Client

After you sign up or log in to PlusAuth, you need to create a client to get the necessary configuration keys in the dashboard.
Go to [Clients](https://dashboard.plusauth.com/~clients) and create a client with the type of `Regular Web Application`


## Configure Client
### Get Client Properties

You will need your `Client Id` and `Client Secret` for interacting with PlusAuth. You can retrieve them from the created client's details.

### Configure Redirect and Logout URIs
When PlusAuth authenticates a user, it needs a URI to redirect back. That URI must be in your client's `Redirect URI` 
list. If your application uses a redirect URI that is not white-listed in your PlusAuth Client, you will receive an error.

The same thing applies to the logout URIs. After the user logs out, you need a URI to be redirected.

::alert{type=info}

If you are following the sample project, the Redirect URL you need to add to the Redirect URIs field is *`http://localhost:3000/login.php`* and 
the Logout URL you need to add to the Post Logout Redirect URIs field is *`http://localhost:3000/`*.
::

## Configure PHP to use PlusAuth
Create a PHP application or download the sample project from the link on the top of the page.

### Install Dependencies

Install following dependencies using `composer` or any other dependency management tool.

```json
// composer.json

{
	"require": {
		"jumbojett/openid-connect-php": "^0.9.5",
		"vlucas/phpdotenv": "^5.0"
	}
}
```

::alert{type=warning}
We are using `phpdotenv` to loads environment variables from `.env` file. DO NOT use this library in `production`
::

### Create the .env file

Create the `.env` file in the root of your app and add your PlusAuth variables and values to it. If you're following the sample project, rename `.env.example` to `.env` and replace the values accordingly.

```properties
# .env
PLUSAUTH_ISSUER_URL=YOUR_PLUSAUTH_DOMAIN
PLUSAUTH_CLIENT_ID=YOUR_CLIENT_ID
PLUSAUTH_CLIENT_SECRET=YOUR_CLIENT_SECRET
```

::alert{type=warning}
Do not put the `.env` file into source control. Otherwise, your history will contain references to your client's secret.
If you are using git, create a `.gitignore` file (or edit your existing one, if you have one already) and add `.env` to it. The `.gitignore` file tells source control to ignore the files (or file patterns) you list. Be careful to add `.env` to your `.gitignore` file and commit that change before you add your `.env`
::

```properties
# .gitignore
.env
```

### Configure OpenIdConnect Client

To enable authentication with PlusAuth, create `auth.php` in `public` and add the following section to your file.

```php
// public/auth.php

<?php 
require_once('../vendor/autoload.php');
use Jumbojett\OpenIDConnectClient;
$dotenv = Dotenv\Dotenv::createImmutable('../');
$dotenv->load();

class Auth {
    /**
     * @var OpenIDConnectClient oidc client
     */
    private $oidc;
    private $postLogoutRedirectUri;

    public function __construct() {
        $oidc = new OpenIDConnectClient( 
            $_ENV['PLUSAUTH_ISSUER_URL'],
            $_ENV['PLUSAUTH_CLIENT_ID'],
            $_ENV['PLUSAUTH_CLIENT_SECRET']);

        $oidc->setResponseTypes('id_token token');
        $oidc->addScope(array('openid profile'));
        $oidc->setAllowImplicitFlow(true);
        $oidc->addAuthParam(array('response_mode' => 'form_post'));
        // Handle PlusAuth response after login
        $oidc->setRedirectURL('http://localhost:3000/login.php');
        
        // For development mode only
        $oidc->setVerifyHost(false);
        $oidc->setVerifyPeer(false);

        $this->oidc = $oidc; // Crate oidc object at page load
        $this->postLogoutRedirectUri = "http://localhost:3000/";
    }

    public function login() {
        if ($this->isLoggedIn() == false) {
            $this->oidc->authenticate();
            $this->setIdToken($this->oidc->getIdToken());
            $this->setUser($this->oidc->requestUserInfo());
        }
        // User information is in the session if the user logged in
      }

    public function logout() {
        // Clear session, user will still be logged in on PlusAuth
        $idToken = $this->getIdToken();
        unset($_SESSION['idToken']);
        unset($_SESSION['user']);

        // RP initiated logout, user will be logged out from PlusAuth too
        return $this->oidc->signOut($idToken, $this->postLogoutRedirectUri);
    }

    // getters...
}

session_start();
$auth = new Auth();
?>
```

We created `Auth` class to configure `OpenIdConnect` client. The `oidc` client stores all user information in `sessions` 

You may have noticed that the values defined in the [Configure Client](#configure-client) section are used here. If you have used different 
values make sure to update this file accordingly.

## Implement login, user profile, and logout

### Add Login and Logout

Add `login.php` to your application to enable the `Login` action. This page provides `/login` endpoint to your application.

```php
// public/Login.php

<?php 
require_once('./auth.php');

$auth->login();

header('Location: index.php');
die();
?>
```

Add `logout.php` to your application to enable the `Logout` action. This page provides `/logout` endpoint to your application.

```php
// public/logout.php

<?php 
require_once('./auth.php');

$auth->logout();

header('Location: index.php');
die();

?>
```

### Create Views

### Layout

Create `header.php` under `public` folder.

```php
<!-- public/header.php -->

<?php 
require_once('./auth.php');

// Get user info stored in session
$user = $auth->getUser();
?>

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta  name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no"/>
    <title>Plusauth Starter Template</title>
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.0/css/bootstrap.min.css"
      integrity="sha384-9aIt2nRpC12Uk9gS9baDl411NQApFmC26EwAOH8WgZl5MYYxFfc+NcPb1dKGj7Sk"
      crossorigin="anonymous" />
  </head>
  <body>
    <style>
      body {
        padding-top: 5rem;
      }
    </style>
    <nav class="navbar navbar-expand-md navbar-dark bg-dark fixed-top">
      <a class="navbar-brand" href="/">Plusauth Starter</a>
      <div class="collapse navbar-collapse" id="navbarsExampleDefault">
        <ul class="navbar-nav mr-auto"></ul>
        <?php if($auth->isLoggedIn()) { ?>
        <li class="nav-item navbar-nav">
          <a class="nav-link" href="/profile.php">
            Logged in as: <?php echo $user->username ?>
          </a>
        </li>
        <a class="nav-link" href="/logout.php">
          Logout
        </a>
        <?php } else { ?>
        <li class="nav-item navbar-nav">
          <a class="nav-link" href="/login.php">
            Login
          </a>
        </li>
        <?php } ?>
      </div>
    </nav>
    <main role="main" class="container">
```

Create `footer.php` under `public` folder.

```html
<!-- public/footer.php -->

</main>
</body>
</html>
```

### Homepage
Create `index.php` under `public` folder.

```php
<!-- public/index.php -->

<?php 
require_once('./auth.php');
?>

<?php include "./header.php" ?>
<div class="jumbotron">
  <div class="container">
    <h1 class="display-3">Hello, world!</h1>
    <p>
      This is a template for a simple login/register system. It includes a
      simple OpenID Connect Implicit Flow. To view Profile page please login.
    </p>
    <p>      
      <?php if ($auth->isLoggedIn()) { ?>
      <a class="btn btn-success btn-lg" href="/profile.php" role="button"
        >View Profile &raquo;</a
      >
      <?php } else { ?>
      <a class="btn btn-primary btn-lg" href="/login.php" role="button"
        >Login/Register &raquo;</a
      >
      <?php } ?>
    </p>
    <p></p>
  </div>
</div>
<?php include "./footer.php" ?>
```

::alert{type=info}

 If a user is not logged in, then the *`Login`* button is shown, else the *`View Profile`* button is. 
::

### User Profile
Create `profile.php` under `public` folder.

```php
<!-- public/profile.php -->

<?php 
require_once('./auth.php');

// Check if user logged in
$auth->login();

$user = $auth->getUser();
?>
<?php include './header.php';  ?>
<div class="container">
  <h3>Welcome <?php echo $user->username; ?>!</h3>
  <h4>User object:</h4>
  <pre> <?php echo print_r($user); ?></pre>
</div>
<?php include './footer.php' ?>
```

## See it in action

That is it. Start your app and point your browser to [http://localhost:300](http://localhost:3000). Follow the **Log In** link to log in or sign up to your PlusAuth tenant. Upon successful login or signup, you should be redirected back to the application.
