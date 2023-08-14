---
title: "Single Responsibility and Decorator Pattern"
date: 2023-08-14T16:46:46-03:00
summary: Using the 'Decorator Pattern' to add new features in a principled way
draft: false
---

## Introduction

Not long ago I was working on a piece of code that had to perform a series of HTTP requests, measure their execution time, and then yield some metrics. In particular, requests should contain an authentication header, and in this case we were using JSON Web Tokens. The details about JWT are irrelevant to the program, except for the fact that the token used was generated from a user provided secret.

On a first design, I decided to create an interface for the `AuthToken` generation. This would allow me to have at least two instances: one would actually generate a valid JWT from a user-provided secret, and the other would return, for example, a fixed token, which is very useful during testing:

```C#
interface IAuth
{
    string AuthToken { get; }
}
```

As you can see, the interface is very minimal with only one method. We fix the type of the token to a `string` since it's good enough for our needs; no need to parametrize in this scenario. For the implementations then we have:

```C#
class FixedAuth : IAuth
{
    public string AuthToken { get; }

    public FixedAuth(string authToken)
    {
        AuthToken = authToken;
    }
}
```

```C#
class JwtAuth : IAuth
{
    private readonly ISystemClock _clock;
    private readonly byte[] _secret;
    private readonly Lazy<string> _token;

    public string AuthToken
    {
        get => _token.Value;
    }

    public JwtAuth(ISystemClock clock, byte[] secret)
    {
        _clock = clock;
        _secret = secret;
        _token = new Lazy<string>(GenerateAuthToken);
    }

    private string GenerateAuthToken()
    {
        var signingKey = new SymmetricSecurityKey(_secret);
        var credentials = new SigningCredentials(signingKey, SecurityAlgorithms.HmacSha256);
        var claims = new[] { new Claim("iat", _clock.UtcNow.ToUnixTimeSeconds().ToString()) };
        var token = new JwtSecurityToken(claims: claims, signingCredentials: credentials);
        var handler = new JwtSecurityTokenHandler();

        return handler.WriteToken(token);
    }
}
```

As you can see, the real implementation (`JwtAuth`) has a lot more moving parts, and generating the actual token is a non-trivial task (note that we even rely on the current time).

Since the token needed to be used multiple times in a very short span of time, my first idea was to cache it using a [`Lazy`](https://learn.microsoft.com/en-us/dotnet/api/system.lazy-1?view=net-7.0) so it would only be computed once.

**Fast forward a couple of days and I get a report that we're getting authentication errors on this piece of code: it seems like the token is only valid for 60 seconds, while we're sending more and more request for a running time well over the 10 minute mark.**

## The wrong fix: Modify the code

My initial thought was to change the code on `JwtAuth` removing the `Lazy` and instead manually controlling when to produce a new token and when to use a previously computed value. After writing the required code I realized that this was **a bad idea** since now the class now had two responsibilities: handling the token generation and also dealing with caching and expirations.

- If the logic for generating the token were to change, we would need to change `JwtAuth` again.
- If the timeout logic were to change, we would need to change `JwtAuth` again.

This is a clear violation of the [Single Responsibility Principle](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html) - the S in "SOLID" - and it usually results in code that is hard to maintain, modify and extend in the long run.

For example, with this approach I could not test the timeout logic in isolation since it's now part of the "real" `IAuth` implementation, `JwtAuth`. Ideally, I should be able to test this logic along side an `IAuth` implementation that, for example, on each call returns a different token value from a `List`. Then, I would verify that each value of this `List` was used as long as the expiration was not reached.

In particular, my first mistake was to add the caching behavior through `Lazy` to the original code.

## The right fix: Use a decorator

After realizing the problem, I removed all usages of `Lazy` from the `JwtAuth` class, which resulted in the following code:

```C#
class JwtAuth : IAuth
{
    private readonly ISystemClock _clock;
    private readonly byte[] _secret;

    public string AuthToken
    {
        get => GenerateAuthToken();
    }

    public JwtAuth(ISystemClock clock, byte[] secret)
    {
        _clock = clock;
        _secret = secret;
    }

    private string GenerateAuthToken()
    {
        var signingKey = new SymmetricSecurityKey(_secret);
        var credentials = new SigningCredentials(signingKey, SecurityAlgorithms.HmacSha256);
        var claims = new[] { new Claim("iat", _clock.UtcNow.ToUnixTimeSeconds().ToString()) };
        var token = new JwtSecurityToken(claims: claims, signingCredentials: credentials);
        var handler = new JwtSecurityTokenHandler();

        return handler.WriteToken(token);
    }
}
```

Now, whenever we use `JwtAuth::AuthToken` we're actually generating a new token through `JwtAuth::GenerateAuthToken`. To get the desired functionality (caching with expiration), I created a new class:

```C#
class TtlAuth : IAuth
{
    private readonly IAuth _auth;
    private readonly ISystemClock _clock;
    private readonly int _ttl;

    private LastAuth? _lastAuth;

    public TtlAuth(IAuth auth, ISystemClock clock, int ttl)
    {
        _auth = auth;
        _clock = clock;
        _ttl = ttl;
    }

    public string AuthToken
    {
        get
        {
            long currentTime = _clock.UtcNow.ToUnixTimeSeconds();
            if (_lastAuth is null || Math.Abs(_lastAuth.GeneratedAt - currentTime) >= _ttl)
            {
                _lastAuth = new(currentTime, _auth.AuthToken);
            }

            return _lastAuth.Token;
        }
    }

    private record LastAuth(long GeneratedAt, string Token);
}
```

This new `TtlAuth` is a [Decorator](https://refactoring.guru/es/design-patterns/decorator). In short, a decorator `D` for an interface `I` is a class that also implements `I` and also has an internal instance of `I`, delegating in some particular way the methods defined in `I`. To be concrete, note the following:


```C#
class TtlAuth : IAuth
```
The class `TtlAuth` implements the interface `IAuth`,


```C#
private readonly IAuth _auth;

public TtlAuth(IAuth auth, ISystemClock clock, int ttl) { /* ... */ }
```
it also has an internal `IAuth` instance.

The delegation here is the interesting part: based on the elapsed time since the last usage of the `AuthToken`, `TtlAuth` will get the token from the underlying `IAuth` or reuse the last value. Note here that `_auth.AuthToken` is actually a function invocation due to the usage of [properties](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/properties) in C#.

The underlying `IAuth` could be anything, from the initial fixed implementation, to the `JwtAuth` we discussed before. But it's important to note that now the logic for caching and expiration is fully self contained on a single class, `TtlAuth`.

Then, in our DI container we can replace which instance we're using for the `IAuth` interface without any dependant code even noticing the change. That is, we go from this:

```C#
collection.AddSingleton<IAuth>(provider =>
    new JwtAuth(
        provider.GetRequiredService<ISystemClock>(),
        secret: config.Secret
    )
);
```

to this:

```C#
collection.AddSingleton<IAuth>(provider =>
    new TtlAuth(
        new JwtAuth(
            provider.GetRequiredService<ISystemClock>(),
            secret: config.Secret
        ),
        provider.GetRequiredService<ISystemClock>(),
        ttl: config.AuthTtl
    )
);
```

## Conclusions

- Whenever you write a piece of code, write the minimum amount of code that gets the feature working. In this case, I should not have added an initial `Lazy` trying to optimize something that I didn't even measure. *Note that if a new token was generated for each request, then the issue with expirations would not have existed on the first place.*

- If you need to extend the code to add new functionality, do not attemp to modify it immediately: try to use a decorator to extend the behavior of the existing code instead. A lot of the times this is actually the case since we need to do something either before or after what the original code was intended to do.

- Code that uses interfaces is easier to test, change and extend: we can create test specific implementations, we can change the implementation without users noticing, and we can add new behavior as long as we don't break the contract established.
