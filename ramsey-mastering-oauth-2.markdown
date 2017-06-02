Mastering OAuth 2.0
-------------------

_June 2, 2017 - Ben Ramsey_

### Introduction

Ben met Cal at OScon 2005 and doesn't feel old.  :)

We're going to deal mostly with client-side, rather than server-side issues.

**Slides:** [bram.se/dc4d-oauth2](bram.se/dc4d-oauth2)

Ben is a "web craftsman, author and speaker." He builds a platform for
photographers at ShootProof and works remotely from Nashville, TN, USA.
He is the co-author of the Zend PHP Certification Study Guide (3rd Ed),
he founded the Atlanta PHP UG and is now involved at Nashville; he
contributed array_column() to the PHP core, and maintains league/oauth2-client
and ramsey/uuid.

### OAuth 2.0 is Not Easy

Everyone has a slightly different implementation. The League of Extraordinary
Packages has come to the rescue with an interoperable library.

OAuth allows you to grant users authorization via access tokens in order to
authenticate in one place via credentials elsewhere on the web.

**Demo - show users their Instagram photos in a new way**

(Code for Ben's demo will be available online)

The demo has a user log in to a dashboard where she then has the option to
authorize Ben's site to access her Instagram account.  Once the Instagram
credentials have been input, Ben is requesting photo access (scope), and
when the user has OK'd this, some magic happens in the background and Ben's
app has direct access to the user's Instagram photos, _even though the user's
credentials are only known to the user and Instagram._

**App Code:** [bram.se/dc4d-oauth2-app](bram.se/dc4d-oauth2-app)

### Preparing for OAuth

1. Register your app with the service
2. Let the service know your domains or redirect URLs
3. Configure your app to use the client ID and secret given by the service

**Example (Instagram)**:

- Sign up for an account
- Head over to API documentation / developer section
    - Manage Clients -> Register New Client (fill out sign-up form)
    - For example / testing, redirect to localhost:8000 URL
    - Take down Client ID / Client Secret, it's basically your user and pass

### Integrating with the Provider

```
composer require league/oauth2-instagram

use League\OAuth2\Client\Provider\Instagram;

$provider = new Instagram([
    // client data here as associative array
]);
```

After this setup, we make an authorization request.

```
$authURL = $provider->getAuthorizationUrl();
$request->session()->put(
    'instagramState',
    $provider->getState()
);
return redirect()->away($authUrl);
```

Then we use the redirection endpoint (receive auth code, check state, and then
exchange the code for an access token).

```
$state = $request->session()->get('instagramState');

// Error handling here

$token = $provider->getAccessToken(
    'authorization_code',
    [
        'code' => $request->code,
    ]
);

$request->session()->put('instagramToken', $token);
return redirect()...
```

Expiration and refreshing of tokens is not provided by Instagram, but we can
handle this on our own by checking the token manually in many cases.

Now we can use the access tokens to get provider data. How?

1. getAuthenticatedRequest() returns a PSR-7 RequestInterface object, OR
2. Use your favorite HTTP request lib (Guzzle) to make a request.

**Example with Guzzle**:

```
$feedRequest = $provider->getAuthenticatedRequest(
    'GET',
    'https://instagram-URL',
    $instagramToken
);

$client = new \GuzzleHttp\Client();
$feedResponse = $client->send($feedRequest);

$instagramFeed = json_decode(
    $feedResponse->getBody()->getContents()
);
```

Remember that OAuth 2.0 is _not a protocol_.  Why bother with this mess?

### A Brief History of Web Authorization

In Web 1.0, we didn't worry about this very much.  There was little shared
data across domains.

In Web 2.0, this is all different.  We did all kinds of mashups to share data
across sites.  Users had to trust sites with sensitive information all over
the place (the _password anti-pattern_).

This made it easy for attackers to take advantage of easy targets.  People tend
to use insecure passwords and use the same passwords across many accounts.  The
sheep are easily fleeced in this system.

### What is OAuth 2.0, then?

It was built to overcome barriers to implementation in OAuth 1.0, because it
had a low adoption rate.

OAuth 2.0 is an _authorization framework_ rather than a protocol (RFC 6749).
Many specifics of implementation are left open, and this has resulted in a
diversity of non-interoperable systems.

But the League library solves this non-interoperability problem. It defines
4 roles:

1. Resource owner (user with data a site wants to access)
2. Resource server (server that hosts user's protected data: Instagram API)
3. Client (app that uses client credentials to request protected data)
4. Authorization server (server that grants access tokens after authentication)

It defines 4 authorization grant types:

It currently defines 4 authorization grant types (league/oauth2-client)

```
// Setting up provider with the generic library:

$provider = new GenericProvider([
    'clientId' => '',
    'clientSecret' => '',
    'redirectUri' => '',
    'urlAuthorize' => '',
    'urlAccessToken' => '',
    'urlResourceOwnerDetails' => 'https://them.example.net/api/me'
]);
```

1. Authorization Code
    - Three-legged: resource owner, client, and auth server
    - Go to client, talk to the auth server, get token, talk to resource server
2. Resource Owner Password Credentials
    - Give username and password to client
    - Client exchanges them for access token
    - USE WITH CAUTION, high trust level of client required (client is OS, Phone)
3. Client Credentials
    - Client is the resource owner (think weather or map data)
    - Credentials are stored in the client
    - $accessToken = `$provider->getAccessToken('client_credentials');`
4. Implicit
    - Relies on client-side redirection using a client ID and known redirection
      URL
    - league/oauth2-client library _cannot support this grant type_
    - Client redirects to provider directly, no access tokens

### Toward a More Secure Web

OAuth 2.0 is only one step on this journey, but it is an important one.
Next steps: see [the slides](bram.se/dc4d-oauth2) for Ben's links - lots
of helpful resources and a couple of books.

There is a _whole lot more_ on the server side (league/oauth2-server).

### Questions

_Should one use timing-safe equality checks to validate the 'state' parameter?_

Most of the time, Ben has seen basic equality used. It's a random string provided
for a single request.  You might want to forget it afterwards.

_Both OAuth and OpenID use tokens... what is the difference between them?_

OpenID provides a federated _login_ system; OAuth 2.0 provides a federated
_authorization_ system.  OpenID seems to be a dormant project, having lost out
to OAuth / 'sign in with Google'.

_Is it necessary or preferable to use a crypto-secured hash for the state
parameter?_

It's a Good Thing to do, but it is not required by OAuth or the library.  Many
providers do not even use the state.

_Is there any way to send access tokens other than query params?_

Ben's recommendation is to use the Authorization HTTP header; there are different
kinds of tokens (look into the 'bearer' token).  You could use encrypted strings
or hashes (even md5 / SHA1); it doesn't need to be crypto-secure.  You could
even define your own token type.

_Do we really need two servers for auth and resource?_

Two requests are necessary, but they _could_ be to the same server, same domain,
etc.

