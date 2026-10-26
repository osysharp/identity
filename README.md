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

## Which roles people are offered, and where they may be held

`app.Roles` lists the roles the app's pages hand out (an invite form, a user manager), how each reads to a person, and
where it may be held — across the app, only inside an organisation, or either (the default):

```osy
app.Roles = new RolesSetup { Offered = [
  new RoleChoice { Role = AppRole.Agent, Label = "Agent", Description = "Works tickets." },
  new RoleChoice { Role = AppRole.Owner, Label = "Owner", Description = "Runs one team.", HeldIn = RoleScope.Organization },
  // Named where somebody holds it, never handed out by a page.
  new RoleChoice { Role = AppRole.Operator, Label = "Operator", Description = "Runs the platform.", Grantable = false },
] };
```

`OfferedRoles(RoleScope.App)` and `IsOfferedRole(r, RoleScope.Organization)` answer for one place; the kits that give
roles offer and give each only where it may be held.

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
`CurrentPrincipal()`, `PersonName(p)` (how a person is named to others — their name, else their address, else `""`
for a system account, so a caller says "someone" rather than leave a blank) and `IsUnreachableAddress(email)` (an
address under the reserved `.invalid` domain: a system account, which is mailed nothing and has no address to prove).

## Building your own on top of it

A package of your own can reference `Principal` and `RoleGrant` like any type, and name the app's roles as `role` —
`role Level;` on an entity, `role.Authenticator` in a rule — which becomes the app's enum when the package is compiled
into it. Fork this repository to change the contract itself.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using IdentityKitTests;

[Test(Club)]
void a_member_signs_in_with_the_apps_own_method() {
  string ticket = "";
  runas (AuthBootstrap) { ticket = SignInWithPin("1001", "4321"); }
  Assert.NotEqual("", ticket);
}
```

From [`tests/identity.test.osy`](tests/identity.test.osy).

```osy
using Osysharp.Identity;
using IdentityKitTests;

[Test(Club)]
void a_wrong_pin_and_an_unknown_number_both_get_nothing() {
  string wrong = "x";
  string unknown = "x";
  runas (AuthBootstrap) { wrong = SignInWithPin("1001", "0000"); unknown = SignInWithPin("9999", "4321"); }
  Assert.Equal("", wrong);
  Assert.Equal("", unknown);
}
```

From [`tests/identity.test.osy`](tests/identity.test.osy).

```osy
using Osysharp.Identity;
using IdentityKitTests;

[Test(Club)]
void a_deactivated_member_cannot_sign_in() {
  runas (AuthBootstrap) { Member.Single(m => m.Number == "1001").IsActive = false; }
  string ticket = "x";
  runas (AuthBootstrap) { ticket = SignInWithPin("1001", "4321"); }
  Assert.Equal("", ticket);
}
```

From [`tests/identity.test.osy`](tests/identity.test.osy).

## Learn more

- [The Osysharp.Identity reference](https://osysharp.com/reference/security/identity-kit/)
- [Osysharp.Identity in the package catalogue](https://osysharp.com/packages/#osysharp-identity)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
