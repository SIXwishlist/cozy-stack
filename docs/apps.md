[Table of contents](README.md#table-of-contents)

# Applications

It's possible to manage serverless applications from the cozy stack and serve
them via cozy stack. The stack does the routing and serve the HTML and the
assets for the applications. The assets of the applications are installed in
the virtual file system.


## Install an application

### The manifest

To install an application, cozy needs a manifest. It's a JSON document that
describes the application (its name and icon for example), how to install it
and what it needs for its usage (the permissions in particular). While we have
considered to use the same [manifest format as the W3C for
PWAs](https://www.w3.org/TR/appmanifest/), it didn't match our expectations.
The [manifest format for
FirefoxOS](https://developer.mozilla.org/en-US/docs/Archive/Firefox_OS/Firefox_OS_apps/Building_apps_for_Firefox_OS/Manifest)
is a better fit. We took a lot of inspirations from it, starting with the
filename for this file: `manifest.webapp`.

Field          | Description
---------------|---------------------------------------------------------------------
name           | the name to display on the home
slug           | the default slug (it can be changed at install time)
editor         | the editor's name to display on the cozy-bar of the app
icon           | an icon for the home
description    | a short description of the application
source         | where the files of the app can be downloaded
developer      | `name` and `url` for the developer
default_locale | the locale used for the name and description fields
locales        | translations of the name and description fields in other locales
version        | the current version number
license        | [the SPDX license identifier](https://spdx.org/licenses/)
intents        | a list of intents provided by this app (see [here](intents.md) for more details)
permissions    | a map of permissions needed by the app (see [here](permissions.md) for more details)
routes         | a map of routes for the app (see below for more details)

### Routes

A route make the mapping between the requested paths and the files. It can
have an index, which is an HTML file, with a token injected on it that
identify both the application. This token must be used with the user cookies
to use the services of the cozy-stack.

By default, a route can be only visited by the authenticated owner of the
instance where the app is installed. But a route can be marked as public.
In that case, anybody can visit the route.

For example, an application can offer an administration interface on `/admin`,
a public page on `/public`, and shared assets in `/assets`:

```json
{
  "/admin": {
    "folder": "/",
    "index": "admin.html",
    "public": false
  },
  "/public": {
    "folder": "/public",
    "index": "index.html",
    "public": true
  },
  "/assets": {
    "folder": "/assets",
    "public": true
  }
}
```

If an application has no routes in its manifest, the stack will create one
route, this default one:

```json
{
  "/": {
    "folder": "/",
    "index": "index.html",
    "public": false
  }
}
```

**Note**: if you have a public route, it's probably better to put the app icon
in it. So, the cozy-bar can display it for the users that go on the public
part of the app.

## Sources

Here is the available sources, defined by the scheme of the source URL:

  - `registry://`: to install an application from the instance registries
  - `git://` or `git+ssh://`: to install an application from a git repository
  - `http://` or `https://`: to install an application from an http server (via a tarball)

The `registry` scheme expect the following elements:

  - scheme: `registry`
  - host: the name of the application
  - path: `/:channel` the channel of the application (see the [registry](docs/registry.md) doc)

Examples: `registry://drive/stable`, `registry://drive/beta`, and `registry://drive/dev`.

For the `git` scheme, the fragment in the URL can be used to specify which branch to install.

For the `http` and `https` schemes, the fragment can be used to give the expected sha256sum.

### POST /apps/:slug

Install an application, ie download the files and put them in `/apps/:slug` in the virtual file system of the user, create an `io.cozy.apps` document, register the permissions, etc.

This endpoint is asynchronous and returns a successful return as soon as the application installation has started, meaning we have successfully reached the manifest and started to fetch application data.

To make this endpoint synchronous, use the header `Accept: text/event-stream`. This will make a eventsource stream sending the manifest and returning when the application has been installed or failed.

#### Status codes

* 202 Accepted, when the application installation has been accepted.
* 400 Bad-Request, when the manifest of the application could not be processed (for instance, it is not valid JSON).
* 404 Not Found, when the manifest or the source of the application is not reachable.
* 422 Unprocessable Entity, when the sent data is invalid (for example, the slug is invalid or the Source parameter is not a proper or supported url)

#### Query-String

Parameter | Description
----------|------------------------------------------------------------
Source    | URL from where the app can be downloaded (only for install)

#### Request

```http
POST /apps/emails?Source=git://github.com/cozy/cozy-emails.git HTTP/1.1
Accept: application/vnd.api+json
```

#### Response

```http
HTTP/1.1 202 Accepted
Content-Type: application/vnd.api+json
```

```json
{
  "data": [{
    "id": "4cfbd8be-8968-11e6-9708-ef55b7c20863",
    "type": "io.cozy.apps",
    "meta": {
      "rev": "1-7a1f918147df94580c92b47275e4604a"
    },
    "attributes": {
      "name": "calendar",
      "state": "installing",
      "slug": "calendar",
      ...
    },
    "links": {
      "self": "/apps/calendar"
    }
  }]
}
```

**Note**: it's possible to choose a git branch by passing it in the fragment
like this:

```http
POST /apps/emails-dev?Source=git://github.com/cozy/cozy-emails.git%23dev HTTP/1.1
```

### PUT /apps/:slug

Update an application with the specified slug name.

This endpoint is asynchronous and returns a successful return as soon as the application installation has started, meaning we have successfully reached the manifest and started to fetch application data.

To make this endpoint synchronous, use the header `Accept: text/event-stream`. This will make a eventsource stream sending the manifest and returning when the application has been updated or failed.

#### Request

```http
PUT /apps/emails HTTP/1.1
Accept: application/vnd.api+json
```

#### Response

```http
HTTP/1.1 202 Accepted
Content-Type: application/vnd.api+json
```

```json
{
  "data": [{
    "id": "4cfbd8be-8968-11e6-9708-ef55b7c20863",
    "type": "io.cozy.apps",
    "meta": {
      "rev": "1-7a1f918147df94580c92b47275e4604a"
    },
    "attributes": {
      "name": "calendar",
      "state": "installing",
      "slug": "calendar",
      ...
    },
    "links": {
      "self": "/apps/calendar"
    }
  }]
}
```

#### Status codes

* 202 Accepted, when the application installation has been accepted.
* 400 Bad-Request, when the manifest of the application could not be processed (for instance, it is not valid JSON).
* 404 Not Found, when the application with the specified slug was not found or when the manifest or the source of the application is not reachable.
* 422 Unprocessable Entity, when the sent data is invalid (for example, the slug is invalid or the Source parameter is not a proper or supported url)

## List installed applications

### GET /apps/

An application can be in one of these states:

- `ready`, the user can use it
- `installing`, the installation is running and the app will soon be usable
- `upgrading`, a new version is being installed
- `uninstalling`, the app will be removed, and will return to the `available` state.
- `errored`, the app is in an error state and can not be used.

#### Request

```http
GET /apps/ HTTP/1.1
Accept: application/vnd.api+json
```

#### Response

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.api+json
```

```json
{
  "data": [{
    "id": "4cfbd8be-8968-11e6-9708-ef55b7c20863",
    "type": "io.cozy.apps",
    "meta": {
      "rev": "2-bbfb0fc32dfcdb5333b28934f195b96a"
    },
    "attributes": {
      "name": "calendar",
      "state": "ready",
      "slug": "calendar",
      ...
    },
    "links": {
      "self": "/apps/calendar",
      "icon": "/apps/calendar/icon",
      "related": "https://calendar.alice.example.com/"
    }
  }]
}
```


## Get the icon of an application

### GET /apps/:slug/icon

#### Request

```http
GET /apps/calendar/icon HTTP/1.1
```

#### Response

```http
HTTP/1.1 200 OK
Content-Type: image/svg+xml
```

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="60" height="60" viewBox="0 0 60 60">
<title>Calendar</title>
<path fill="#fff" fill-opacity=".011" d="M30 58.75c15.878 0 28.75-12.872 28.75-28.75s-12.872-28.75-28.75-28.75-28.75 12.872-28.75 28.75 12.872 28.75 28.75 28.75zm0 1.25c-16.569 0-30-13.432-30-30 0-16.569 13.431-30 30-30 16.568 0 30 13.431 30 30 0 16.568-13.432 30-30 30z"/>
<path d="M47.997 9h-35.993c-1.654 0-3.004 1.345-3.004 3.004v35.993c0 1.653 1.345 3.004 3.004 3.004h35.993c1.653 0 3.004-1.345 3.004-3.004v-35.993c0-1.654-1.345-3.004-3.004-3.004zm-35.997 3.035h4.148v2.257c0 .856.7 1.556 1.556 1.556s1.556-.7 1.556-1.556v-2.257h5.136v2.257c0 .856.7 1.556 1.556 1.556s1.556-.7 1.556-1.556v-2.257h5.137v2.257c0 .856.699 1.556 1.556 1.556s1.556-.7 1.556-1.556v-2.257h5.137v2.257c0 .856.699 1.556 1.556 1.556s1.556-.7 1.556-1.556v-2.257h3.992v6.965h-35.998v-6.965zm36 35.965h-36v-27h36v27zm-21.71-10.15c-.433.34-.997.51-1.69.51-.64 0-1.207-.137-1.7-.409-.493-.273-.933-.603-1.32-.99l-1.1 1.479c.453.508 1.027.934 1.72 1.28.693.347 1.56.521 2.6.521.613 0 1.19-.083 1.73-.25.54-.167 1.013-.407 1.42-.721.407-.312.727-.696.96-1.149.233-.453.35-.974.35-1.56 0-.841-.25-1.527-.75-2.061s-1.13-.9-1.89-1.1v-.08c.693-.268 1.237-.641 1.63-1.12.393-.48.59-1.08.59-1.8 0-.533-.1-1.01-.3-1.43-.2-.42-.48-.772-.84-1.06-.36-.287-.793-.503-1.3-.65-.507-.147-1.067-.22-1.68-.22-.76 0-1.45.147-2.07.44s-1.203.68-1.75 1.16l1.18 1.42c.387-.36.783-.65 1.19-.87.407-.22.863-.33 1.37-.33.587 0 1.047.15 1.38.45.333.3.5.717.5 1.25 0 .293-.057.566-.17.819-.113.253-.3.47-.56.65-.26.18-.6.319-1.02.42-.42.1-.943.149-1.57.149v1.681c.72 0 1.32.05 1.8.149.48.101.863.243 1.15.43.287.188.49.414.61.681.12.267.18.567.18.899 0 .602-.217 1.072-.65 1.412zm13.71.15h-3v-11h-2c-.4.24-.65.723-1.109.89-.461.167-1.25.49-1.891.61v1.5h3v8h-3v2h8v-2z"/>
</svg>
```


## Uninstall an application

### DELETE /apps/:slug

#### Request

```http
DELETE /apps/tasky HTTP/1.1
```

#### Response

```http
HTTP/1.1 204 No Content
```


## Access an application

Each application will run on its sub-domain. The sub-domain is the slug used
when installing the application (`calendar.cozy.example.org` if it was
installed via a POST on /apps/calendar). On the main domain
(`cozy.example.org`), there will be the registration process, the login form,
and it will redirect to `home.cozy.example.org` for logged-in users.

### Rationale

The applications have different roles and permissions. An application is
identified when talking to the stack via a token. The token has to be injected
in the application, one way or another, and it have to be unaccessible from
other apps.

If the applications run on the same domain (for example
`https://cozy.example.org/apps/calendar`), it's nearly impossible to protect
an application to take the token of another application. The security model of
the web is based too heavily on [Same Origin
Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)
for that. If the token is put in the html of the index.html page of an app,
another app can use the fetch API to get the html page, parse its content and
extract the token. We can think of other ways to inject the token (indexeddb,
URL, cookies) but none will provide a better isolation. Maybe in a couple
years, when [the origin spec](https://w3c.github.io/webappsec-suborigins/)
will be more advanced.

So, if having the apps on the same origin is not possible, we have to put them
on several origins. One interesting way to do that is using sandboxed iframes.
An iframe with [the sandbox
attribute](https://www.w3.org/TR/html5/embedded-content-0.html#attr-iframe-sandbox),
and not `allow-same-origin` in it, will be assigned to a unique origin. The
W3C warns:

> Potentially hostile files should not be served from the same server as the
> file containing the iframe element. Sandboxing hostile content is of minimal
> help if an attacker can convince the user to just visit the hostile content
> directly, rather than in the iframe. To limit the damage that can be caused
> by hostile HTML content, it should be served from a separate dedicated
> domain. Using a different domain ensures that scripts in the files are
> unable to attack the site, even if the user is tricked into visiting those
> pages directly, without the protection of the sandbox attribute.

It may be possible to disable all html pages to have an html content-type,
except the home, and having the home loading the apps in a sandboxed iframe,
via the `srcdoc` attribute. But, it will mean that we will have to reinvent
nearly everything. Even showing an image can no longer be done via an `<img>`
tag, it will need to use post-message with the home. Such a solution is
difficult to implement, is a very fragile (both for the apps developer than
for security) and is an hell to debug when it breaks. Clearly, it's not an
acceptable solution.

Thus, the only choice is to have several origins, and sub-domains is the best
way for that. Of course, it has the downside to be more complicated to deploy
(DNS and TLS certificates). But, in the tradeoff between security and ease of
administration, we definetively take the security first.

### Routes

> Should we be concerned that all the routes are on the same sub-domain?

No, it's not an issue. There are two types of routes: the ones that are
publics and those reserved to the authenticated user. Public routes have no
token

Private routes are private, they can be accessed only with a valid session
cookie, ie by the owner of the instance. Another application can't use the
user cookies to read the token, because of the restrictions of the same origin
policy (they are on different domains). And the application can't use an open
proxy to read the private route, because it doesn't have the user cookies
for that (the cookie is marked as `httpOnly`).
