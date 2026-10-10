# Osysharp.Identity

Who signs in to an app, and which of the app's roles each of them holds — the two types every kit that knows about
people builds on. With them in one place, a sign-in kit, an invitations kit and a user manager all mean the same
person, whichever of them the app uses.

- **`Principal`** — the app's one kind of signed-in person: an address (one per account, across every kind of
  account), a display name, whether it may sign in, whether its address was proven, where it comes from.
- **`RoleGrant`** — a role a person holds, as a member of the **app's own** `[Role]` enum.

It has **no password and no pages**. [Osysharp.UserAccounts](https://osyrin.com/templates/kits/user-accounts/) is the default way in, adding
`User : Principal` with a password. An app that signs people in some other way derives its own type from `Principal`.

## Use it

```osy
// app.osy
use Osysharp.Identity@0;
```

```osy
using Osysharp.Identity;

[Role] enum AppRole { Authenticator, Agent, Lead }      // the app's roles; Authenticator is the sign-in flow's
policy IsLead => RoleGrant.Any(g => g.Member == user && g.Role == AppRole.Lead);

// Who may grant roles is the app's decision:
partial entity RoleGrant { security { allow create, delete when IsLead; } }
```

## What every kind of account stands on

The kit's rules hold on every type derived from `Principal`, whatever that type grants itself:

- an account edits its own display name, but its address, whether that address is proven, whether it may sign in,
  where it comes from and when it last signed in are written only by the sign-in flow;
- each person reads their own role grants and nobody else's (the sign-in flow reads them too, to decide what a sign-in
  asks for); only the sign-in flow grants one unless the app says otherwise;
- `user` in every rule is the base `Principal`, and `user is Member` tells the kinds apart.

## Your own way in

```osy
entity Member : Principal {
  [Required, Unique, MaxLength(20)] string Number;
  [MaxLength(200)] string PinHash;
  security {
    allow read when IsAuthenticated || IsAuthenticator;
    allow create when IsAuthenticator;
    allow update when IsAuthenticated || IsAuthenticator where Id == user.Id;
    allow update when IsAuthenticator;
    deny read PinHash when !IsAuthenticator;
  }
}

[AuthMethod]
string SignInWithPin(string number, string pin) {
  var m = Member.Where(x => x.Number == number.Trim()).FirstOrDefault();
  if (m == null || !m.IsActive) { Security.VerifyPassword(pin); return ""; }   // a miss costs the same time
  if (!Security.VerifyPassword(pin, m.PinHash)) { return ""; }
  m.LastLoginAt = DateTime.UtcNow;
  return Security.IssueJwt(m.Id, m.Email);
}

app.AuthBootstrap = new AuthBootstrap { Role = AppRole.Authenticator, Login = SignInWithPin };
```

Helpers: `NormalisedEmail(email)` (trimmed, lower-case — how addresses are stored and compared), `LooksLikeEmail(email)`,
`CurrentPrincipal()`.

## Building your own on top of it

A package of your own can reference `Principal` and `RoleGrant` like any type, and name the app's roles as `role` —
`role Level;` on an entity, `role.Authenticator` in a rule — which becomes the app's enum when the package is compiled
into it. Fork this repository to change the contract itself.
