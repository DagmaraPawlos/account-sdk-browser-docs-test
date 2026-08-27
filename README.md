![Build Status](https://schibsted.ghe.com/user-identity/account-sdk-browser/actions/workflows/pr.yml/badge.svg)
[![Code coverage](https://codecov.io/gh/schibsted/account-sdk-browser/branch/master/graph/badge.svg)](https://codecov.io/gh/schibsted/account-sdk-browser)
[![Snyk](https://snyk.io/test/github/schibsted/account-sdk-browser/badge.svg?targetFile=package.json)](https://snyk.io/test/github/schibsted/account-sdk-browser)

# Schibsted Account SDK for browsers

Welcome! This is the home of the Schibsted Account JavaScript SDK for browsers. Use it to let users
sign up and log in with Schibsted Account, read the current session, and check access to products
and features.

The source code is maintained in a private Schibsted repository. Public API documentation is
available at
[schibsted.github.io/account-sdk-browser](https://schibsted.github.io/account-sdk-browser/).

## Public API

Create one SDK instance with `createAccountSdk()`. Use `getAccountSdk()` when another module needs
to wait for that initialized `Account` instance.

The package publishes ES modules and modern JavaScript, including async/await and WHATWG browser
APIs. It does not include polyfills.

## Installation

```sh
npm install --save @schibsted/account-sdk-browser
```

```js
import { createAccountSdk, getAccountSdk } from '@schibsted/account-sdk-browser';
```

## Before You Start

The SDK communicates mainly with Session Service on a brand domain such as `id.vg.no`. Your website
and Session Service domain must share the same top-level site so browser cookies are sent with XHR
requests.

For local development:

- serve your site over HTTPS, because Session Service is hosted over HTTPS
- use a local hostname with the same top-level site as the configured Session Service domain

For example, if your pre-production site is `pre.sdk-example.com` and uses
`id.pre.sdk-example.com` as Session Service, use a local hostname such as
`local.sdk-example.com`.

If this is a new site and you do not have a `sessionDomain` yet, contact
[support](mailto:schibstedaccount@schibsted.com).

## Initialize the SDK

Create the SDK once during application startup:

```js
import { createAccountSdk } from '@schibsted/account-sdk-browser';

const account = createAccountSdk({
    clientId: '56e9a5d1eee0000000000000',
    redirectUri: 'https://awesomenews.site/callback',
    sessionDomain: 'https://id.awesomenews.site',
    env: 'PRE',
    defaultProductIds: ['product-id'],
});
```

Required configuration:

- `clientId`: your Schibsted Account client id
- `redirectUri`: the callback URL registered for your client
- `sessionDomain`: your client-configured Session Service domain

Optional configuration:

- `env`: defaults to `PRE`; supported keys include `LOCAL`, `DEV`, `PRE`, `PRO`, `PRO_NO`,
  `PRO_FI`, and `PRO_DK`
- `callbackBeforeRedirect`: called before a full-page session refresh redirect
- `defaultProductIds`: product or feature ids used when `hasAccess()` is called without arguments
- `log`: receives SDK debug log lines
- `varnish`: enables Varnish cookie handling
- `window`: overrides the browser window object, mostly useful in tests

`createAccountSdk()` registers the instance on `window.schAccount` and emits
`schAccount:ready` event.

```js
window.addEventListener('schAccount:ready', (event) => {
    const account = event.detail.instance;
});
```

If another module needs the initialized instance, use `getAccountSdk()`:

```js
import { getAccountSdk } from '@schibsted/account-sdk-browser';

const account = await getAccountSdk();
```

`getAccountSdk()` resolves immediately if the SDK has already been initialized. Otherwise it waits
for `schAccount:ready` and rejects if initialization does not happen within 4000 ms.

## Basic Usage

After initialization, use the `Account` instance to render user state and start login when needed:

```js
async function renderLoginState() {
    const container = document.getElementById('login-container');

    if (await account.isConnected()) {
        const user = await account.getUser();
        container.textContent = `Hello ${user.givenName || user.displayName || user.userId}`;
        return;
    }

    container.innerHTML = '<button type="button">Log in</button>';
    container.querySelector('button').addEventListener('click', () => {
        account.login({ state: createLoginState() });
    });
}
```

## Login Flow

Use `account.login()` to start the OAuth authorization flow:

```js
account.login({
    state: createLoginState(),
    scope: 'openid',
    preferPopup: false,
});
```

By default the SDK redirects the current window. Set `preferPopup: true` when login is triggered by
a user gesture and you want to try a popup first. If the popup cannot be opened, the SDK falls back
to the redirect flow.

If you only need the URL, use `account.loginUrl()`:

```js
const url = account.loginUrl({ state: createLoginState() });
```

### Handling `state`

`state` is an opaque OpenID Connect value that is returned to your `redirectUri` with the
authorization `code`. Use it to protect against CSRF and to restore application state.

A common pattern is:

1. Your backend creates a random token and stores it temporarily.
2. Your frontend calls `account.login({ state })` with that token, or with an encoded payload that
   contains it.
3. Schibsted Account redirects back to your `redirectUri` with `code` and `state`.
4. Your backend validates the returned `state` before exchanging `code` for tokens.

Keep Access Tokens and Refresh Tokens on your backend. Do not send them to the browser.

### Authentication Methods

Use `acrValues` when you need to request a specific authentication method:

```js
account.login({
    state: createLoginState(),
    acrValues: 'otp-email',
});
```

Supported values include `password`, `otp`, `sms`, `otp-email`, `eid`, `eid-no`, `eid-se`,
`eid-fi`, and `eid-dk`. Values other than `otp-email` can be combined as a space-separated string
where supported by the account flow. Verify the resulting AMR claim in the ID token on your backend
if you need to enforce a completed authentication method.

## Session and User Data

Use these methods to inspect the current Session Service state:

- `account.hasSession()` returns the raw SDK-known session response
- `account.isLoggedIn()` returns whether the browser has a Schibsted Account session
- `account.isConnected()` returns whether the user is connected to your client
- `account.getUser()` returns the connected user session data
- `account.getUserId()` returns the realm-specific user id as a number
- `account.getExternalId()` returns an identifier specific to the configured client and supplied
  external party; this method is deprecated and retained only for compatibility with existing
  integrations
- `account.getUserUuid()` returns the globally unique user id
- `account.getUserSDRN()` returns the user SDRN
- `account.getSpId()` returns the Varnish `sp_id` value when present

Some session methods can trigger a full-page Session Service refresh redirect, especially for
Safari-based browsers. Use `callbackBeforeRedirect` if you need to persist client state before that
happens.

```js
const account = createAccountSdk({
    clientId,
    redirectUri,
    sessionDomain,
    callbackBeforeRedirect: () => {
        saveCurrentUiState();
    },
});
```

## Logout and Account Page

Use `account.logout()` to remove the brand Session Service session and redirect the browser:

```js
account.logout('https://awesomenews.site/logged-out');
```

Use `account.logoutUrl()` when you need the logout URL without navigating immediately:

```js
const logoutUrl = account.logoutUrl('https://awesomenews.site/logged-out');
```

Use `account.accountUrl()` to send the user to their Schibsted Account profile page:

```js
window.location.href = account.accountUrl('https://awesomenews.site/account-return');
```

## Access Checks

Use `account.hasAccess()` to check access to Schibsted Account product ids or Zuora feature ids:

```js
const access = await account.hasAccess();

if (access?.entitled) {
    showPremiumContent();
} else {
    showPaywall();
}
```

Pass product ids to override the configured defaults for a specific check:

```js
const access = await account.hasAccess(['other-product-id']);
```

`hasAccess()` requires `sessionDomain`. Results are cached according to the Session Service TTL.
If no product ids are passed to the method, `defaultProductIds` must be configured. Use
`account.clearCachedAccessResult(productIds, userId)` when you need to invalidate that cache.

## Simplified Login Widget

Before using the simplified login widget, make sure your site has no site-specific terms and
conditions in the Schibsted Account login flow.

The widget is shown only when the SDK can read enough user context from the global account session.
Your application decides when and how often to show it.

```js
if (!(await account.isConnected()) && shouldShowSimplifiedLogin()) {
    const opened = await account.showSimplifiedLoginWidget(
        { state: createLoginState },
        { locale: 'nb' },
    );

    if (opened) {
        rememberSimplifiedLoginWasShown();
    }
}
```

`state` may be a string or a function that returns a string or `Promise<string>`. The function is
called only if the user continues from the widget.

## Events

The SDK exposes fluent `on()` and `off()` aliases for browser-native event listeners.
Payload-bearing events are `CustomEvent` instances and expose their payload through `detail`:

```ts
import type { AccountEventListener } from '@schibsted/account-sdk-browser';

const handleLogin: AccountEventListener<'login'> = (event) => {
    console.log('Login started', event.detail.url, event.detail.method);
};

account.on('login', handleLogin);
account.off('login', handleLogin);
```

Available Account events:

- `login` — a `CustomEvent` whose `detail` contains `url` and the `method` (`popup` or `default`)
- `logout` — a `CustomEvent` whose `detail` contains the logout `url`
- `simplifiedLoginOpened` — an `Event` emitted when the widget is displayed
- `simplifiedLoginCancelled` — an `Event` emitted when the widget is closed

The SDK also emits `schAccount:ready` on `window` when an Account instance is created.

## Errors

Validation, network, and service failures from API calls reject or throw `SDKError`. Convenience
methods such as `isLoggedIn()`, `isConnected()`, and `getSpId()` return fallback values instead of
throwing when the session lookup fails.

```js
import { SDKError } from '@schibsted/account-sdk-browser';

try {
    await account.getUser();
} catch (error) {
    if (error instanceof SDKError) {
        console.error(error.toString());
    }
}
```

## API Documentation

See the [public API documentation](https://schibsted.github.io/account-sdk-browser/) for the API
matching the latest published package. Use the version selector in the documentation when your
application uses an older SDK version.

## Documentation Publishing

The documentation workflow runs when a GitHub Release is published. It builds the documentation
from that release's exact tag and publishes it to the public
[`schibsted/account-sdk-browser`](https://github.com/schibsted/account-sdk-browser) repository
without removing documentation versions already stored there.

Each release is available at its own versioned URL. The documentation root, version selector, and
public repository README are updated to the latest published release. An existing release can also
be published or rebuilt by running the workflow manually with its tag.

Publishing requires the `public-docs` environment and a `PUBLIC_GITHUB_DOCS_TOKEN` secret with
write access to the public repository.

Run `npm run docs:preview` to build the current documentation in the versioned layout and preview
it locally over HTTP.

## Publishing

Versioning and changelog updates are handled by
[Release Please](https://github.com/googleapis/release-please). The release configuration lives in
[`release-please-config.json`](./release-please-config.json), and publishing to
[npmjs.org](https://www.npmjs.com/package/@schibsted/account-sdk-browser) is handled by the
[npm-publish workflow](./.github/workflows/npm-publish.yml) when a GitHub Release is published.

Use [Conventional Commits](https://www.conventionalcommits.org/) for changes that should be picked
up by release automation.

## Example Project

A live example integration is available at
[pro.sdk-example.com](https://pro.sdk-example.com).

## License

Copyright (c) 2026 Schibsted Products & Technology AS

Licensed under the
[MIT License](https://github.com/schibsted/account-sdk-browser/blob/master/LICENSE.md).
