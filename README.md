# Steady Provider for OAuth 2.0 Client

This package provides Steady OAuth 2.0 support for the PHP League's [OAuth 2.0 Client](https://github.com/thephpleague/oauth2-client).

[![CircleCI](https://circleci.com/gh/oliverschloebe/oauth2-steadyhq/tree/master.svg?style=svg)](https://circleci.com/gh/oliverschloebe/oauth2-steadyhq/tree/master) [![Build Status](https://travis-ci.com/oliverschloebe/oauth2-steadyhq.svg?branch=master)](https://travis-ci.com/oliverschloebe/oauth2-steadyhq)
[![StyleCI](https://github.styleci.io/repos/189750356/shield?branch=master)](https://github.styleci.io/repos/189750356)
[![License](https://img.shields.io/packagist/l/oliverschloebe/oauth2-steadyhq.svg)](https://github.com/oliverschloebe/oauth2-steadyhq/blob/master/LICENSE)
[![Latest Stable Version](https://img.shields.io/packagist/v/oliverschloebe/oauth2-steadyhq.svg)](https://packagist.org/packages/oliverschloebe/oauth2-steadyhq)
[![Latest Version](https://img.shields.io/github/release/oliverschloebe/oauth2-steadyhq.svg?style=flat-square)](https://github.com/oliverschloebe/oauth2-steadyhq/releases)
[![Source Code](https://img.shields.io/badge/source-oliverschloebe/oauth2--steadyhq-blue.svg?style=flat-square)](https://github.com/oliverschloebe/oauth2-steadyhq)

---

## Installation

```
composer require oliverschloebe/oauth2-steadyhq
```

## Steady API

https://developers.steadyhq.com/#oauth-2-0

## Usage

[Register your apps on steadyhq.com](https://steadyhq.com/de/backend/) to get `clientId` and `clientSecret`.

### OAuth2 Authentication Flow

```php
require_once __DIR__ . '/vendor/autoload.php';

//session_start(); // optional, depending on your used data store

$steadyProvider = new \OliverSchloebe\OAuth2\Client\Provider\Steady([
	'clientId'	=> 'yourId',          // The client ID of your Steady app
	'clientSecret'	=> 'yourSecret',      // The client password of your Steady app
	'redirectUri'	=> 'yourRedirectUri'  // The return URL you specified for your app on Steady
]);

// Get authorization code
if (!isset($_GET['code'])) {
	// Options are optional, scope defaults to ['read']
	$options = [ 'scope' => ['read'] ];
	// Get authorization URL
	$authorizationUrl = $steadyProvider->getAuthorizationUrl($options);

	// Get state and store it to the session
	$_SESSION['oauth2state'] = $steadyProvider->getState();

	// Redirect user to authorization URL
	header('Location: ' . $authorizationUrl);
	exit;
} elseif (empty($_GET['state']) || (isset($_SESSION['oauth2state']) && $_GET['state'] !== $_SESSION['oauth2state'])) { // Check for errors
	if (isset($_SESSION['oauth2state'])) {
		unset($_SESSION['oauth2state']);
	}
	exit('Invalid state');
} else {
	// Get access token
	try {
		$accessToken = $steadyProvider->getAccessToken(
			'authorization_code',
			[
				'code' => $_GET['code']
			]
		);
		
		// We have an access token, which we may use in authenticated
		// requests against the Steady API.
		echo 'Access Token: ' . $accessToken->getToken() . "<br />";
		echo 'Refresh Token: ' . $accessToken->getRefreshToken() . "<br />";
		echo 'Expired in: ' . $accessToken->getExpires() . "<br />";
		echo 'Already expired? ' . ($accessToken->hasExpired() ? 'expired' : 'not expired') . "<br />";
	} catch (\League\OAuth2\Client\Provider\Exception\IdentityProviderException $e) {
		exit($e->getMessage());
	}

	// Get resource owner
	try {
		$resourceOwner = $steadyProvider->getResourceOwner($accessToken);
	} catch (\League\OAuth2\Client\Provider\Exception\IdentityProviderException $e) {
		exit($e->getMessage());
	}
        
	// Store the results to session or whatever
	$_SESSION['accessToken'] = $accessToken;
	$_SESSION['resourceOwner'] = $resourceOwner;
    
	var_dump(
		$resourceOwner->getId(),
		$resourceOwner->getName(),
		$resourceOwner->getEmail(),
		$resourceOwner->getAttribute('email'), // allows dot notation, e.g. $resourceOwner->getAttribute('group.field')
		$resourceOwner->toArray()
	);
}
```

### Refreshing a Token

Once your application is authorized, you can refresh an expired token using a refresh token rather than going through the entire process of obtaining a brand new token. To do so, simply reuse this refresh token from your data store to request a refresh.

```php
$steadyProvider = new \OliverSchloebe\OAuth2\Client\Provider\Steady([
	'clientId'	=> 'yourId',          // The client ID of your Steady app
	'clientSecret'	=> 'yourSecret',      // The client password of your Steady app
	'redirectUri'	=> 'yourRedirectUri'  // The return URL you specified for your app on Steady
]);

$existingAccessToken = getAccessTokenFromYourDataStore();

if ($existingAccessToken->hasExpired()) {
	$newAccessToken = $steadyProvider->getAccessToken('refresh_token', [
		'refresh_token' => $existingAccessToken->getRefreshToken()
	]);

	// Purge old access token and store new access token to your data store.
}
```

### Sending authenticated API requests

The Steady OAuth 2.0 provider provides a way to get an authenticated API request for the service, using the access token; it returns an object conforming to `Psr\Http\Message\RequestInterface`.

```php
$mySubscriptionsRequest = $steadyProvider->getAuthenticatedRequest(
	'GET',
	'https://steadyhq.com/api/v1/subscriptions/me', // see https://developers.steadyhq.com/#current-subscription
	$accessToken
);

// Get parsed response of current authenticated user's subscriptions; returns array|mixed
$mySubscriptions = $steadyProvider->getParsedResponse($mySubscriptionsRequest);

var_dump($mySubscriptions);
```

### Sending non-authenticated API requests

Send a non-authenticated API request to public endpoints of the Steady API; it returns an object conforming to `Psr\Http\Message\RequestInterface`.

```php
$subscriptionsParams = [ 'filter[subscriber][email]' => 'alice@example.com' ];
$subscriptionsRequest = $steadyProvider->getRequest(
	'GET',
	'https://steadyhq.com/api/v1/subscriptions?' . http_build_query($subscriptionsParams)
);

// Get parsed response of non-authenticated API request; returns array|mixed
$subscriptions = $steadyProvider->getParsedResponse($subscriptionsRequest);

var_dump($subscriptions);
```

### Using a proxy

It is possible to use a proxy to debug HTTP calls made to Steady. All you need to do is set the proxy and verify options when creating your Steady OAuth2 instance. Make sure to enable SSL proxying in your proxy.

```php
$steadyProvider = new \OliverSchloebe\OAuth2\Client\Provider\Steady([
	'clientId'	=> 'yourId',          // The client ID of your Steady app
	'clientSecret'	=> 'yourSecret',      // The client password of your Steady app
	'redirectUri'	=> 'yourRedirectUri'  // The return URL you specified for your app on Steady
	'proxy'		=> '192.168.0.1:8888',
	'verify'	=> false
]);
```

## Requirements

PHP 8.0 or higher.

## Testing

``` bash
$ ./vendor/bin/parallel-lint src test
$ ./vendor/bin/phpunit --coverage-text
$ ./vendor/bin/phpcs src --standard=psr2 -sp
```

## License

The MIT License (MIT). Please see [License File](https://github.com/oliverschloebe/oauth2-steadyhq/blob/master/LICENSE) for more information.
