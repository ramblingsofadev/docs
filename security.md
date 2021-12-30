<h1 id="doc-title">Security</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Introduction](#introduction)
    1. [Example](#example)
2. [Working With Principals](#working-with-principals)

</div>

</nav>

<h2 id="introduction">Introduction</h2>

The security library adds concepts for managing principals, identities, and claims for use in [authentication](authentication.md) and [authorization](authorization.md).  A principal is the thing, usually a user or system process, that interacts with your application.  You can authenticate a principal and authorize actions performed by it.  A principal contains one or more identities, each of which may contain claims.  Claims are statements about an identity, eg email, date of birth, or roles.

It should be noted that, in Aphiria, a principal is a generic representation of a user, and is solely meant for authorization and authentication.  Your application will likely have its own abstractions for users, and those abstractions' data can be used to populate identities and claims of a principal.

<h3 id="example">Example</h3>

Let's say you're going to the airport to board a flight.  You, the principal, will be asked to prove you are who you claim to be, which you'll do with your passport - an identity that contains claims about your citizenship, name, and date of birth.  At security, you'll be asked to re-prove your identity with your passport, and also be asked to prove that you have a ticket - another identity that contains claims about your name, your flight number, and assigned seat.  Airport security will authenticate your passport and ticket, and verify that you're authorized to board the flight.

<h2 id="working-with-principals">Working With Principals</h2>

To create a principal, you first create its claims for its identities.  Let's look at an example:

```php
use Aphiria\Security\{Claim, ClaimType, Identity, User};

// Claims data is usually stored in a database
$claimsIssuer = 'example.com';
// Claim types may either be ClaimType enum values or your own custom strings
$claims = [
    // This claim stores the user's ID
    new Claim(ClaimType::NameIdentifier, '123', $claimsIssuer),
    // This claim stores the user's name
    new Claim(ClaimType::Name, 'Dave', $claimsIssuer),
    // This claim stores the user's roles
    new Claim(ClaimType::Role, 'admin', $claimsIssuer)
];
// You can also pass in an array of identities
$user = new User(new Identity($claims));
```

> **Note:** You can have multiple claims of the same type in one identity.  For example, you might have multiple roles for a user, each of which would have its own distinct claim.

`User` contains some useful methods for aggregating claims made by all its identities:

```php
$allClaims = $user->getClaims();
$allRoleClaims = $user->getClaims(ClaimType::Role);

if ($user->hasClaim(ClaimType::Role, 'admin')) {
    // ...
}

// Add another identity
$identity = new Identity([new Claim(ClaimType::Email, 'foo@example.com', 'example.com')]);
$user->addIdentity($identity);
```

You can also loop through all of the user's identities and query them directly:

```php
foreach ($user->getIdentities() as $identity) {
    $allClaims = $identity->getClaims();
    $allRoleClaims = $identity->getClaims(ClaimType::Role);
    
    if ($identity->hasClaim(ClaimType::Role, 'admin')) {
        // ...
    }
    
    // A convenience method for grabbing this identity's username
    $username = $identity->getName();
    // Another convenience method for grabbing this identity's ID
    $id = $identity->getNameIdentifier();
}
```

You'll frequently find yourself dealing with the user's primary identity.  By default, this is the first identity added to the user, but it can be customized with a callback to determine the primary identity.

```php
// This will make the last added identity the primary one
$primaryIdentitySelector = fn (array $identities): ?IIdentity => $identities[\count($identities) - 1] ?? null;
$user = new User($claims, $primaryIdentitySelector);
$userId = $user->getPrimaryIdentity()?->getNameIdentifier();
```

Typically, the process of authentication will create the principal and store it, usually as a [request](http-requests.md) property.  To learn more, read the [authentication documentation](authentication.md).
