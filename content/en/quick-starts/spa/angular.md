---
title: Angular Tutorial
description: This tutorial demonstrates how to add user login, logout, and profile to an Angular application.
topics:
  - quickstarts
  - single-page-application
  - login
  - user profile
  - logout
  - angular
  - angular13
type: spa
icon: i-devicon-angular
label: Angular
---

This tutorial shows how to use PlusAuth with [Angular](https://angular.io/) Single Page Application. If you do not have a PlusAuth account, register from [here](https://dashboard.plusauth.com).

::alert{type=info}
This tutorial follows [plusauth-angular-starter](https://github.com/PlusAuth/plusauth-angular-starter) sample project on Github. You can download and follow the tutorial via the sample project.
::

## Create PlusAuth Client

After you sign up or log in to PlusAuth, you need to create a client to get the necessary configuration keys in the dashboard.
Go to [Clients](https://dashboard.plusauth.com/~clients) and create a client with the type of `Single Page Application`


## Configure Client
### Get Client Properties

You will need your `Client Id` for interacting with PlusAuth. You can retrieve it from the created client's details.

### Configure Redirect and Logout URIs

When PlusAuth authenticates a user, it needs a URI to redirect back with access and id token. That URI must be in your client's `Redirect URI` 
list. If your application uses a redirect URI which is not white-listed in your PlusAuth Client, you will receive an error.

The same thing applies to the logout URIs. After the user logs out, you need a URI to be redirected.

::alert{type=info}
If you are following the sample project, the Redirect URL you need to add to the Redirect URIs fields are *`http://localhost:4200/callback`* and *`http://localhost:4200/silent-renew`*.
The Logout URL you need to add to the Post Logout Redirect URIs field is *`http://localhost:4200/`*.
::


## Create an Angular Application

Install the Angular CLI globally using `npm` and create a new Angular project. Add a router to the project to render different views.

```bash
# Create the application using the ng new.
ng new plusauth-angular-starter --routing

# Move into the project directory
cd plusauth-angular-starter

# Add bootstrap for styling
npm install bootstrap
```

::alert{type=info}
The `ng new` command prompts you for information about features to include in the initial application project. `--routing` flag adds the routing by default.
::

### Install OIDC Client
For interacting with PlusAuth it is advised to use an OpenID Connect library.
In this tutorial we will be using [oidc-client-js](https://github.com/PlusAuth/oidc-client-js) 
but you could use any OpenID Connect library.

Install `oidc-client-js` with the following command
```bash
npm install @plusauth/oidc-client-js
```

::alert{type=info}
`oidc-client-js` is an OpenID Connect (OIDC) and OAuth2 library for browser based JavaScript applications.
You can find source code on [Github](https://github.com/PlusAuth/oidc-client-js) and the API documentation [here](https://docs.plusauth.com/tools/oidc-client-js).
::

## Configure Angular Application to use PlusAuth

We will be using `environment` files for maintaining providing some constant values.

### Edit environment.ts file

Edit the `environment.ts` file in `src/environments` with the following and modify values accordingly.
```typescript
// environment.ts

export const environment = {
  production: false,
  oidcIssuer: 'https://<YOUR_PLUSAUTH_TENANT_NAME>.plusauth.com/',
  clientId: '<YOUR_PLUSAUTH_CLIENT_ID>'
};
```

::alert{type=info}
If you are following the sample project, rename `environment.example.ts` to `environment.ts` and replace the values accordingly.
::

### Configure Angular Application
Let's start by editing our application's entry point module.
Edit `app.module.ts` in `src/app` folder. Import `components` and other `modules` that we will create later.

```ts
// app.module.ts

import { ApplicationRef, DoBootstrap, NgModule } from '@angular/core'
import { BrowserModule } from '@angular/platform-browser'

import { AppRoutingModule } from './app-routing.module'
import { AppComponent } from './app.component'
import { HeaderComponent } from './components/header.component'
import { HomeComponent } from './views/home.component'
import { SilentRenewComponent } from './components/silent-renew.component'
import { AuthCallbackComponent } from './components/auth-callback.component'
import { AuthService } from './services/auth.service'
import { ProfileComponent } from './views/profile.component'
import { UnauthorizedComponent } from './views/unauthorized.component'

@NgModule({
  declarations: [
    AppComponent,
    HeaderComponent,
    HomeComponent,
    SilentRenewComponent,
    AuthCallbackComponent,
    ProfileComponent,
    UnauthorizedComponent
  ],
  imports: [
    BrowserModule,
    AppRoutingModule
  ],
  providers: [AuthService]   // Make AuthService injected globally
})
export class AppModule implements DoBootstrap {

  constructor(private authService: AuthService) { } // inject auth service

  ngDoBootstrap(appRef: ApplicationRef) {
    // First initialize OIDC Client
    this.authService.getClient().initialize().then(() => {
      appRef.bootstrap(AppComponent)  // Then bootstrap App component
    })
      .catch(console.error)
  }
}
```

::alert{type=info}
`AuthService` is provided in `AppModule` that makes it injectable globally. `ngDoBootstrap` is used to bootstrap `AppComponent` after `AuthService` is initialized.
::

### Configure OIDC Client
We need to initialize our OIDC Client library to handle authentication-related operations.
Create `auth.service.ts` in `src/app/services` folder. Configure `oidc-client-js` as following:

```ts
// auth.service.ts

import { Injectable } from '@angular/core'
import { OIDCClient } from '@plusauth/oidc-client-js'
import { environment } from '../../environments/environment'

@Injectable({
  providedIn: 'root'
})
// Singleton AuthService
export class AuthService {

  private oidcClient
  private user: any = null

  constructor() {
    this.oidcClient = this.getOIDCClient()
    this.oidcClient.on('user_login', user => this.updateUser(user))
    this.oidcClient.on('user_logout', () => this.updateUser(null))
  }

  async getIsLoggedIn(): Promise<any> {
    return await this.oidcClient.isLoggedIn(true)
  }

  getClient(): OIDCClient {
    return this.oidcClient
  }

  getUser(): any {
    return this.user
  }

  updateUser(user: any): void {
    this.user = user
  }

  login(): void {
    this.oidcClient.login()
  }

  logout(): void {
    this.oidcClient.logout()
  }

  async loginCallback() {
    await this.oidcClient.loginCallback(window.location.href)
  }

  getOIDCClient(): OIDCClient {
    return new OIDCClient({
      issuer: environment.oidcIssuer,
      client_id: environment.clientId,
      redirect_uri: 'http://localhost:4200/callback',
      response_mode: 'form_post',
      response_type: 'id_token token',
      post_logout_redirect_uri: 'http://localhost:4200/',
      autoSilentRenew: true,
      checkSession: true,
      requestUserInfo: true,
      scope: 'openid profile',
      silent_redirect_uri: 'http://localhost:4200/silent-renew'
    })
  }

}
```

You may have noticed that the values defined in the [Configure Client](#configure-client) section are used here. If you have used different 
values make sure to update this file accordingly.

### Configure Router
Now let's define our application's router module. We are going to define the routes of our views. 

Create `app-routing.module.ts` in `src/app` folder as following:

```ts
// app-routing.module.ts

import { NgModule } from '@angular/core'
import { RouterModule, Routes } from '@angular/router'
import { AuthCallbackComponent } from './components/auth-callback.component'
import { HomeComponent } from './views/home.component'
import { ProfileComponent } from './views/profile.component'
import { AuthGuardService } from './services/auth-guard.service'
import { SilentRenewComponent } from './components/silent-renew.component'
import { UnauthorizedComponent } from './views/unauthorized.component'

const routes: Routes = [
    { path: 'profile', component: ProfileComponent, canActivate: [AuthGuardService] },
    { path: 'unauthorized', component: UnauthorizedComponent }, // Page if user not authorized
    { path: 'silent-renew', component: SilentRenewComponent },  // Token silent renew uri
    { path: 'callback', component: AuthCallbackComponent },     // Authentication redirect uri
    { path: '**', component: HomeComponent }                
]

@NgModule({
    imports: [RouterModule.forRoot(routes)],
    exports: [RouterModule]
})
export class AppRoutingModule { }
```

`AuthGuardService` guard service will ensure the related route is activated only by authenticated users.
Now create `auth-guard.service.ts` in `src/app/services` as following:

```ts
// auth-guard.service.ts

import { Injectable } from '@angular/core'
import { CanActivate, Router } from '@angular/router'
import { AuthService } from './auth.service'

@Injectable({
  providedIn: 'root'
})
export class AuthGuardService implements CanActivate {

  constructor(private authService: AuthService, private router: Router) { }

  // Check user if logged in for routes that requires auth
  async canActivate(): Promise<boolean> {
    if (await this.authService.getIsLoggedIn()) {
      return true
    }
    this.router.navigate(['/unauthorized'])
    return false
  }
}
```

## Implement login, user profile, and logout

Until now, we have defined our authentication helper and routes. It is time to create the pages and interact with auth helper.

### Edit Main Layout Component
Let's create a simple layout for our application. Add `app-header` and `router-outlet` to `app.component.ts` in `src/app`.

```ts
// app.component.ts

import { Component } from '@angular/core'

@Component({
  selector: 'app-root',
  template: `
    <app-header></app-header>

    <router-outlet></router-outlet>
  `,
  styleUrls: []
})
export class AppComponent {
  title = 'plusauth-angular-starter';
}

```

### Create Header Component

Create `header.component.ts` under `src/app/components` folder. It will be a basic header. If a user is authenticated, it will show the user's identifier and a `Logout` button. 
If not, a `Login` button will be there to initiate login. 

```ts
// header.component.ts

import { Component } from '@angular/core'
import { AuthService } from '../services/auth.service'

@Component({
  selector: 'app-header',
  template: `
    <header>
      <nav class="navbar navbar-expand-md navbar-dark bg-dark fixed-top">
        <div class="container-fluid">
          <a class="navbar-brand" href="/">PlusAuth Starter</a>
          <ul class="nav navbar-nav navbar-right">
            <li
              *ngIf="authService.getUser(); else notLogged"
              class="nav-item navbar-nav"
            >
              <a class="nav-link" [routerLink]="['/profile']"
                >Logged in as: {{ authService.getUser().user.username }}</a
              >
              <button class="btn btn-link" (click)="authService.logout()">
                Logout
              </button>
            </li>
            <ng-template #notLogged>
              <li class="nav-item navbar-nav">
                <button class="btn btn-link" (click)="authService.login()">
                  Login
                </button>
              </li>
            </ng-template>
          </ul>
        </div>
      </nav>
    </header>
  `,
  styleUrls: []
})
export class HeaderComponent {

  constructor(public authService: AuthService) { }

}
```

### Create AuthCallback 
To handle authorization results after a successful login, we need
a simple page and let the library handle the authentication result.
Create `auth-callback.component.ts` under `src/app/components` folder.

```ts
// auth-callback.component.ts

import { Component, OnInit } from '@angular/core'
import { Router } from '@angular/router'
import { AuthService } from '../services/auth.service'

@Component({
  selector: 'app-auth-callback',
  template: `<p>Auth Callback in progress..</p>`,
  styleUrls: []
})
export class AuthCallbackComponent implements OnInit {

  constructor(private authService: AuthService, private router: Router) { }

  async ngOnInit(): Promise<void> {
    try {
        await this.authService.loginCallback()
        this.router.navigate(['/'])
    } catch (e) {
        console.error(e)
    }
  }

}
```

### Create SilentRenew
Access tokens retrieved from PlusAuth have a life span.
`oidc-client-js` automatically provides `access_token` renewal without too much hassle.
Before your access token expires, it will receive a new one in the background so that 
your users will have a flawless app experience without signing in again.

Create `silent-renew.component.ts` under `src/app/components` folder as following:

```ts
// silent-renew.component.ts

import { Component, OnInit } from '@angular/core'
import { AuthService } from '../services/auth.service'

@Component({
  selector: 'app-silent-renew',
  template: `<p></p>`,
  styleUrls: []
})
export class SilentRenewComponent implements OnInit {

  constructor(private authService: AuthService) { }

  ngOnInit() {
    this.authService.getClient().loginCallback()
  }

}
```

### Create Views
### HomePage

Create `home.component.ts` under `src/app/views`.

```ts
// home.component.ts

import { Component } from '@angular/core'
import { AuthService } from '../services/auth.service'

@Component({
  selector: 'app-home',
  template: `
    <div class="jumbotron">
      <div class="container">
        <h1 class="display-3">Hello, world!</h1>
        <p>
          This is a template for a simple login/register system. It includes the
          OpenID Connect Implicit Flow. To view Profile page please login.
        </p>
        <p>
          <a
            [routerLink]="['/profile']"
            class="btn btn-success btn-lg"
            *ngIf="authService.getUser(); else notLogged"
          >
            View Profile &raquo;
          </a>
          <ng-template #notLogged>
            <button class="btn btn-primary btn-lg" (click)="authService.login()">
              Login/Register &raquo;
            </button>
          </ng-template>
        </p>
      </div>
    </div>
  `,
  styleUrls: []
})
export class HomeComponent {

  constructor(public authService: AuthService) { }

}
```

### Profile Page

Create `profile.component.ts` under `src/app/views`.

```ts
// profile.component.ts

import { Component } from '@angular/core'
import { AuthService } from '../services/auth.service'

@Component({
  selector: 'app-profile',
  template: `
    <div class="container" v-if="user">
      <h3>Welcome {{ authService.getUser().user.username }} !</h3>
      <pre>User object: {{ getUserString() }} </pre>
    </div>
  `,
  styleUrls: []
})
export class ProfileComponent {

  constructor(public authService: AuthService) { }

  getUserString(): string {
    return JSON.stringify(this.authService.getUser().user, null, 2)
  }

}
```

### Add Unauthorized Page
We will display a page whenever a user tries to access a protected route without signing in.

Create `unauthorized.component.ts` under `src/app/views`.

```ts
// unauthorized.component.ts

import { Component } from '@angular/core'
import { AuthService } from '../services/auth.service'

@Component({
  selector: 'app-unauthorized',
  templateUrl: `
    <div class="container">
      <p>You must log in to view the page</p>
      <button class="btn btn-primary" (click)="authService.login()">Log in</button>
    </div>
  `,
  styleUrls: []
})
export class UnauthorizedComponent {

  constructor(public authService: AuthService) { }

}
```

## See it in action

That's it. Start your app and point your browser to [http://localhost:4200](http://localhost:4200). Follow the **Log In** link to log in or sign up to your PlusAuth tenant. Upon successful login or signup, you should be redirected back to the application.
