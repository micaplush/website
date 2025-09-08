---
title: Notes from trying a YubiKey for a week
date: 2025-09-08T00:00:00Z
description:
  Evaluating a YubiKey for my own use cases and comparing it with alternatives
  from Token2
---

I was thinking about getting a hardware security token (the thingies with FIDO2
and such) and I was lent a YubiKey 5 NFC to play around with. This post explains
my considerations and findings from that test.

{{< sidenote >}}

{{< reactiontext reaction="laugh/453d" >}}

**Fair warning:** Everything I'm doing is overkill. Normal people do not need a
threat model this paranoid and a TOTP app is probably enough.

{{</ reactiontext >}}

{{</ sidenote >}}

## Security architecture goals

To be able to assess how useful a hardware security token would be for me, I've
defined some high-level goals for my personal security architecture (how I do
things related to security). Some of them are "must-haves", meaning that these
are goals that can't be compromised on if I were to get a hardware token while
some of them are "nice-to-haves" meaning that such a goal isn't necessary but it
could be a tiebreaker if two products came out head-to-head.

**Must have: Avoid having secrets served by "agents" running locally without any
user interaction.** This includes SSH agent and GPG agent on my laptop. If
someone were to compromise my laptop, that would help with limiting lateral
spread to other machines, though it's not a panacea. A sufficiently advanced
attacker could lay dormant until I'm about to initialize an SSH session manually
and hijack that. Even in that case however, I would probably notice it and not
open new SSH sessions, meaning lateral movement is still limited.

**Must have: Avoid having secrets sit on the filesystem unencrypted.** This is
similar to the former goal and on my laptop, this currently includes
[age](https://github.com/FiloSottile/age) private keys.

**Nice to have:** Move highly sensitive secrets such as admin private keys for
my entire infrastructure onto a dedicated hardware device. For high-risk
secrets, this makes it easier to keep track of where they are and which systems
have access to them.

**Nice to have:** FIDO could save time over opening an authenticator app and
typing a TOTP code.

**Nice to have:** Phishing resistance on the second authentication factor for
online services.

**Nice to have:** Having the option to use passkeys in a format that works
cross-platform and is secure. macOS does support passkeys natively but none of
the browsers I use integrate with that. Additionally, I want to preserve the
option to switch back to Linux, meaning my only real option is storing passkeys
in a hardware token (or not using them).

{{< sidenote >}}

{{< reactiontext reaction="annoyed/f5a9" >}}

My password manager on desktop computers, KeePassXC, also supports storing
passkeys. However, its Android counterpart, KeePassDX, doesn't seem to support
that.

{{</ reactiontext >}}

{{< reactiontext reaction="annoyed/0786" >}}

Additionally, I have some reservations against making a local password manager
my sole authentication factor. Especially on Linux (but also on macOS to a
limited degree), local apps can often fake input events, meaning they could in
theory automate interacting with the password manager to export passkeys. But
that varies between distros and configurations.

{{</ reactiontext >}}

{{</ sidenote >}}

**Nice to have:** FIDO2 Level 2 certification, which is often required in
contexts such as banking or government services. I don't think my bank supports
FIDO keys in any way but my home country does have a pretty good e-government
system which requires FIDO2 Level 2 certification on hardware tokens. (And it
allowlists specific models on the backend which is annoying.)

**Must not: Rely on being in possession of a physical item to be able to
authenticate or use secrets.** A backup authentication method that's easy to
replicate and store (in encrypted form) offsite is a must. If my house burns
down and I lose both the primary and the backup token, I don't wanna be locked
out of all of my online accounts.

**Must not: Have a secret or online account compromised if someone else comes in
possession of a hardware token.** A chance of compromise that is highly unlikely
to occur (e.g. an attacker randomly guesses the correct PIN in 3 attempts) is
acceptable.

## Goals of the test

In my tests with the YubiKey, I set out to verify if my goals are realistic and
therefore, if I should get a security token for myself and determine the minimum
viable feature set a hardware token would need to support for my use cases, in
order to decide between two vendors.

## Competing vendors

These are the two competing vendors I narrowed it down to. Other vendors either
didn't seem to offer better pricing or the desired features.

### Yubico

{{< figure src="./yubikey5.webp" alt="A product photo of a YubiKey 5 NFC with a USB C connector" caption="A YubiKey 5 NFC. In case you were wondering how a Yubico Security Key looks, it's exactly the same. Really makes you wonder if it's the same hardware and they're just charging a €30 premium for unlocking more firmware features..." >}}

The YubiKey 5 series has the most features but is also very expensive. Supported
features include:

- FIDO2
- Acting as an OpenPGP SmartCard
- PIV (PKCS#11 SmartCard functionality)
- OTP codes generated on-device
- Proprietary extra auth mechanisms with two programmable "slots"

Products from the Security Key series are pure FIDO2 tokens and more reasonably
priced (though still more expensive than the competition).

Products from the YubiKey Bio Series are also pure FIDO2 tokens with biometric
authentication but they're prohibitively expensive (upwards of €100 per token
which is insane).

Every Yubico client app or tool I've encountered in this test was open source.
That's a huge plus.

The firmware on the keys is not open source. That's obviously a downside but it
should be noted that the firmware on a YubiKey is not replaceable or upgradable
anyways. This is a common practice in the industry to avoid security issues from
an insecure upgrade mechanism.

### Token2

The PIN+ Series has a feature set that sits somewhere between a YubiKey 5 and
Yubico Security Keys. It supports FIDO2, OpenPGP, and OTP codes generated
on-device.

The PIN+Bio3 token adds biometric authentication at a reasonable price (being
just slightly more expensive than Yubico Security Key).

Client apps and tools for Token2 products are not uniformly open source. The
fido2-manage tool, which is used to manage passkeys on desktop devices is open
source (though the interface for FIDO seems to be standardized anyways, so
clients for this feature should be interchangeable). Everything involving OTP
seems to be source-available (to paying customers) but proprietary.

{{< sidenote >}}

{{< reactiontext reaction="annoyed/3200" >}}

They're using a license called the "Fair Source License" (version 0.9) for their
proprietary tools but, at the time of writing, they don't link the correct
license text anywhere on their website.

The initiative behind this license seems to be dead, so the only copy I could
find was on the Internet Archive. Worse, the domain linked on the Token2 website
that was previously related to the Fair Source License has changed hands and is
now associated with a different "Fair Source" effort.

I've notified Token2 about this issue and they've acknowledged it.

{{</ reactiontext >}}

{{</ sidenote >}}

Token2's firmware is at least partially open source (there's a
[firmware repository on GitHub](https://github.com/token2/pin_plus_firmware)).
It's not entirely clear to me if that includes all of the firmware or just a
subset. FIDO2 functionality is definitely included, but I'm not sure about
OpenPGP and OTP. My guess is that those are in separate Java applets which are
not open source.

Similar to a YubiKey, the firmware on a Token2 key cannot be replaced, to avoid
security issues.

## Test methodology

All tests were conducted on macOS Sequoia 15.6.1.

### SSH

SSH is a must-have feature. I've tried different methods of authenticating using
a hardware token figuring out the benefits and drawbacks of each:

- OpenSSH's FIDO2 support
- YubiKey PIV with SSH certificates
- Using a PGP authentication key

### PGP

PGP is a must-have feature. GnuPG only really works with compatible smartcards
and those are very standardized so I just tried if it works in general and
checked if it's painful to use for example when signing multiple Git commits.

### age

Protecting age keys is a must-have feature. I've tested two popular plugins to
make age work with hardware tokens figuring out the benefits and drawbacks of
each:

- [age-plugin-yubikey](https://github.com/str4d/age-plugin-yubikey)
- [age-plugin-fido2-hmac](https://github.com/olastor/age-plugin-fido2-hmac/)

### FIDO2 as a second factor

Using FIDO2 as a second factor is not strictly a must-have but I would probably
think over buying a hardware token again if it turns out it doesn't work (since
that's kinda their main point).

The test methodology for this feature was simple:

- Set up a token as a second factor on my GitHub account
- Check if I can log in on Tor Browser (my daily driver) and Librewolf
- Check if I can log in on GrapheneOS using Tor Browser and Vanadium

### Passkeys

Passkeys are entirely optional but they could be nice if they work and they
don't introduce new risks. If I think the authentication or backup methods are
insufficient, I'll just not use them.

The tests were the same as with FIDO2 as a second factor. Additionally I made
sure it requires some other form of authentication beyond possession of the
token.

### Additional tests

I tested the OTP capabilities of the YubiKey and the integration of YubiKeys in
KeePassXC and KeePassDX. These tests didn't influence my buying decisions but
it's nice to try things out while I have access to a YubiKey.

## Findings

### SSH

#### FIDO2 authentication

The
[setup guide from Yubico](https://developers.yubico.com/SSH/Securing_SSH_with_FIDO2.html)
is mostly correct. Not mentioned in the guide:

- `ssh-askpass` needs to be installed
- SSH agent needs to be started with the following environment variables:
  - `SSH_ASKPASS=$(which ssh-askpass)`
  - `DISPLAY=localhost:0.0`

The article leaves it unclear if the FIDO PIN can be cached if you set it up
according to its instructions. It cannot. Generating a key with
`-O resident -O verify-required` asks for the PIN every time the key is used.

SSH agent does not seem necessary in this case. The "private key" file in the
client is just a pointer to the hardware token, so it doesn't need to be
protected with a password.

This setup requires OpenSSH from Homebrew (or other sources which are close to
upstream). Apple's OpenSSH build won't work. That means keychain integration on
macOS goes out the window (upstream OpenSSH does not support that).

In practice a PIN and touch are required for every login which is cumbersome.
However, biometrics might be supported for this use case. Fingerprint auth could
make this a lot less painful.

Generating the SSH key with `-O credProtect` could be a viable alternative. In
this mode, the private key material is stored in the actual private key file and
it's encrypted using a key stored on the hardware token. That means you need
both the hardware token, and the key file to use the SSH key. I wasn't able to
try this out since the firmware on the YubiKey I had didn't support his feature
but newer YubiKeys supposedly do.

{{< sidenote >}}

{{< reactiontext reaction="annoyed/f5a9" >}}

Yubico has suspiciously
[removed mentions of `credProtect`](https://github.com/Yubico/developers.yubico.com/commit/ae784d9f9fba24f38e4a61f0d58ae3c4db8a1c8c)
from their guide while I was drafting this post. Sooo... this option would
probably require more research.

{{</ reactiontext >}}

{{</ sidenote >}}

There's also the alternative of not requiring a PIN and keeping the private key
resident on the hardware token but I'm not sure if that's secure under my thread
model. It DOES require the "private key" stub on the client and a PIN seems to
be required to export that from the hardware token but I'm not entirely sure if
that's meaningful as a security barrier. Yubico discourages using this method in
their guide.

#### YubiKey PIV authentication

The
[setup guide from Yubico](https://developers.yubico.com/PIV/Guides/SSH_user_certificates.html)
is very outdated. It mentions `yubico-piv-tool` which has been deprecated and no
longer works with the YubiKey I've tested it with. Instead you're supposed to
use `ykman` (commonly part of the `yubikey-manager` package).
[Commands for `ykman`](https://github.com/Yubico/yubico-piv-tool/issues/158#issuecomment-1212158063)
are described in a GitHub issue comment written by a community member. Yikes!

When using Nix to set things up, the relevant dylib for PIV integration in
OpenSSH can be found in the `yubico-piv-tool` package of all places:

```nix
${pkgs.yubico-piv-tool}/lib/libykcs11.dylib
```

To start the SSH agent:

```sh
ssh-agent -P '/nix/store/<hash>-yubico-piv-tool-2.7.1/lib/*'
```

To add the YubiKey to the agent (which is required before SSHing into anything):

```sh
ssh-add -s /nix/store/<hash>-yubico-piv-tool-2.7.1/lib/libykcs11.dylib
```

A PIN is required when adding the token to the agent. For logging into an SSH
server, touching the token is enough. Using the token for FIDO2 authentication
doesn't cause the PIN to be requested again.

A major downside: Only one SSH key can be stored this way.

Cancelling an SSH login while the YubiKey is awaiting touch confirmation leaves
the touch request on the YubiKey (i.e. YubiKey is still waiting for a touch).
This doesn't break anything but it's kind of a weird state. You can either touch
the key or unplug and re-plug it at this point.

Cancelling PIN entry cancels that SSH authentication method and doesn't leave
the token in a weird state.

Unplugging and re-plugging the YubiKey does not require you to enter a PIN again
which means the PIN is probably cached in the SSH agent. (Not sure what to make
of that in terms of security but I guess it's fine...?)

#### GPG for SSH authentication

Turns out GnuPG can act as an SSH agent and authenticate to SSH servers using a
PGP key with authentication capabilities.

Yubico's docs link to a _third party guide_ (basically just someone's blog)
which doesn't go in-depth about how this setup works.

Essentially: A YubiKey has one slot for a PGP key that can do authentication.
This slot has to be filled, then GnuPG can just use it. It doesn't matter if you
generate the whole key at once or add an authentication subkey to an existing
PGP key later.

The UX for generating an authentication subkey is very weird. You have to invoke
GnuPG like so:

```sh
gpg2 --expert --edit-key BLABLABLA
```

Using `gpg` instead of `gpg2` or leaving out the `--expert` flag for some reason
hid everything related to authentication from me. When it asks for the key's
purpose select the option that says something along the lines of "any purpose",
then it lets you choose options related to authentication.

Using PGP has largely the same benefits and drawbacks as the PIV method:

- The PIN is required only once
- Touch is required for every login
- It doesn't require PIN entry again after doing FIDO stuff in a browser
- Only one SSH key can be stored this way

One benefit over the PIV method: This would probably also work fine on the
Token2 hardware tokens. OpenPGP is standardized so I'm fairly confident this
works. Maybe the semantics around switching between PGP and other applets on the
token are different (requiring PIN entry more often).

Using this method requires you to use GnuPG's agent as an SSH agent. It can
store other SSH keys like normal so this doesn't seem like a drawback compared
to using the upstream OpenSSH agent which can't talk to macOS keychain either.
Maybe GPG even has a Keychain integration, I haven't checked.

Otherwise it's very similar to the PIV method: Cancelling an SSH login while
awaiting touch leaves the touch request on the YubiKey, cancelling PIN entry
cancels that SSH authentication method and doesn't leave the token in a weird
state.

Removing and re-plugging the YubiKey does require the PIN again so the PIN is
not cached anywhere external to the token.

#### Verdict

**FIDO2 authentication would be preferred.** It can store multiple identities
and it works with standard OpenSSH utilities (except for Apple's fork).

PIN entry needs to be solved. Biometrics look promising for that. A random blog
post I found
[mentions that OpenSSH supports biometrics](https://swjm.blog/the-complete-guide-to-ssh-with-fido2-security-keys-841063a04252)
for FIDO2 authentication.

**PGP can be used as an alternative.** Since the interface is so standardized, I
expect this to work with Token2 as well. It can only store one key, so less
frequently used keys could still use FIDO2 as an authentication method. (FIDO2
doesn't require support from the agent, so that should be fine.)

**YubiKey PIV has no real benefits over OpenPGP** other than leaving the OpenPGP
authentication slot free for other use cases.

SSH would likely work with both vendors. Token2 might have an edge due to the
possibility to use biometrics with FIDO2 authentication.

### PGP

Works as expected. I would expect either vendor to handle this well since the
interface seems very standardized.

The Gentoo wiki has a good page on it: https://wiki.gentoo.org/wiki/YubiKey/GPG.

PIN entry semantics are already explained in the SSH section above. Touch
policies are configurable (i.e. if touch is required for encryption, signing, or
authentication). Requiring touch every time is probably the most secure option.
I'm not sure if that's a standardized feature or proprietary to YubiKeys.

{{< sidenote >}}

{{< reactiontext reaction="disappointed/f080" >}}

An open question: What would I do if I were to lose the hardware token that
stores my PGP keys?

The PGP module locks up on too many incorrect authentication attempts. I could
assume that there is no other way to extract key data and thus, the key wasn't
compromised (unless the attacker manages to guess a PIN which is highly
unlikely). Or I could rotate my PGP keys which is a bit annoying.

FIDO2 and other authentication mechanisms on the token don't usually have this
issue since it's easy to remove them as authentication methods.

{{</ reactiontext >}}

{{</ sidenote >}}

#### age

Most of the secrets related to my infrastructure are stored in age files. I've
written a tool that generates age secrets automatically and keeps them up to
date. Secrets are defined as declarative templates and can include stuff like
references to other secrets. The generator decrypts every secret, checks if it
still matches the template and updates secrets as needed.

That means the generator has to decrypt many age files at once. (At the time of
writing my infrastructure monorepo contains 142 age files.) Touching the
hardware token for every file would not be feasible.

Sometimes I edit age files manually using the `agenix` CLI (a NixOS-adjacent
tool) or decrypt them using the `age` CLI directly. These workflows should also
be supported somehow.

#### age-plugin-yubikey

This plugin has configurable PIN and touch policies. The one that's useful for
my use case:

- PIN policy "ONCE"
- Touch policy "CACHED"

With this setup, a PIN is required once per session (generally until the key is
unplugged or used for something else). Touching the key is required for
decryption, and is cached for 15 seconds.

This plugin seamlessly integrates with my workflow and the UX is pretty good.
However, the way I've set it up, it does come with the disadvantage of letting
any locally running app use the key for 15 seconds after touching the token.

#### age-plugin-fido2-hmac

Requiring a PIN to decrypt data is optional but necessitated by my thread model.
If configured that way, PIN entry and touching the token are required for every
decryption operation, nothing is cached.

Out of the box that would be unsuitable for my use case but I think I can modify
my tooling to make it work.

- Instead of encrypting infra secrets with the public key of the token, encrypt
  them with the public key of a regular age identity, call that identity the
  "inner identity"
- Take the inner identity's private key and encrypt with the public key of the
  token
- Store the encrypted private key somewhere on the filesystem or in a keychain
- The secrets generator first obtains the inner identity by decrypting it using
  the token
- It keeps the inner identity's private key in memory until it's done (if
  possible marking the memory region as sensitive so that it's not written to
  nonvolatile storage)
- Other tools like `agenix` need to be wrapped in a script to decrypt the inner
  identity as needed, or reimplemented in Go to be able to keep the inner
  identity's private key in memory

A downside of this workaround: The key that ultimately decrypts infra secrets is
handled outside of the hardware token. However, it should be possible to keep
the inner identity's private key completely in memory and thus inaccessible to
other applications at all times. That's better than the 15 second window where
any application can decrypt sensitive data.

#### Verdict

A YubiKey obviously offers more features in this regard but a FIDO2 key would
probably also suffice and using FIDO2 would entail some benefits of its own.

{{< sidenote >}}

{{< reactiontext reaction="laugh/bf0c" >}}

Side note: Token2 is looking to add PIV functionality to its keys in an upcoming
firmware release! Version 3.3 is supposed to have it.

{{</ reactiontext >}}

{{< reactiontext reaction="excited/5bd5" >}}

I've heard through the grapevine (on fedi) that they're expecting to release
this version in Q4 of 2025. The initial release is expected to provide PIV
functionality only to Windows clients though.

age-plugin-yubikey will very likely not work out of the box but perhaps
something similar could be implemented for Token2's PIV applet.

{{</ reactiontext >}}

{{</ sidenote >}}

### FIDO2 as a second factor

#### On desktop

**Tor Browser** requires the user pref `security.webauth.webauthn` to be flipped
from `false` to `true`. That _is_ a possible fingerprinting vector, however: I
use separate browser instances for general browsing (where privacy is more
important) and logging into accounts, so that's not a big deal.

A real risk with Tor Browser: Will this feature remain working? The pref is
disabled by default and there's an open issue in the Android release to remove a
dependency required for WebAuthn.

**Librewolf** works without issue.

#### On GrapheneOS (Android)

FIDO2 on Android relies on a provider that is installed as an app. In most
phones, that's tied into Google Services Framework (GSF). I don't have that on
my phone.

An alternative open source provider exists:
https://codeberg.org/s1m/hw-fido2-provider. It's built on code from the microG
project and it seemed to work fine in my tests. How well it continues to work in
the long term remains to be seen.

**Tor Browser** requires the same user pref modification as on desktop. At the
time of writing it relies on a proprietary Google library to implement FIDO2
authentication.

There are efforts to remove that dependency (though they don't seem very
active):
https://gitlab.torproject.org/tpo/applications/tor-browser/-/issues/41161.

There are similarly inactive efforts to replace it with an open source library:
https://gitlab.torproject.org/tpo/applications/tor-browser/-/issues/41162.

Having a separate browser instance is not as straightforward on Android but the
Alpha release app could double as a "login-only" instance with the pref changed
while the stable release app stays unchanged.

**Vanadium** works without issue once a FIDO2 provider is installed.

### Passkeys

Passkeys work fine with GitHub.

- A PIN is required for every login
- Multiple passkeys can be configured
- Other authentication methods such as TOTP can also be configured
- There's also one-time recovery keys

All in all, this seems acceptable.

I've also tried the [Pocket ID demo](https://demo.pocket-id.org/start-demo)
(because I've been eyeballing that thing for SSO in my personal infra).

- A PIN is required for every login
- Multiple passkeys can be configured
- The backup method here is "log into the server via SSH and run a command"
  which is a bit whack

Not sure about this one yet but it's good to know that apparently every service
requires a PIN for passkeys.

{{< sidenote >}}

{{< reactiontext reaction="nerd-think/0261" >}}

I wonder if tokens with fingerprint sensors can use those instead of asking for
a PIN.

That would be acceptable within my threat model I think and it would bump the
comfiness factor a fair bit.

{{</ reactiontext >}}

{{</ sidenote >}}

### Additional tests

YubiKey OTP works as expected, no surprises there.

KeePassXC and KeePassDX both work nicely with the challenge-response
authentication mechanism proprietary to YubiKeys but I won't be using that
feature.

It _requires_ a YubiKey to unlock the KeePass database.

The secret programmed into the YubiKey _can_ be backed up outside of the token
but in order to execute the challenge, a YubiKey programmed with the same secret
is required.

No backup method can be registered since the challenge response generated by the
YubiKey is used in the database decryption process. If I lost all of my
YubiKeys, I wouldn't have access to my KeePass database until I've received a
replacement.

### Additional considerations

Yubico's client apps and tools seem to be nicer compared to Token2's.

- All of the relevant tools I've encountered were open source
- Yubico Authenticator has a pleasant look and feel across multiple platforms

Token2's apps aren't uniformly open source. The Android companion app and
desktop OTP tooling are relevant for me but proprietary.

The Token2 Android companion app is distributed via Google Play and as an APK
file on their website. I don't use Google Play, so I'll have to go the APK
route. I'm not sure if there's any auto-update mechanism baked into the APK
distribution.

{{< sidenote >}}

{{< reactiontext reaction="angy/3a64" >}}

It would be **so** nice if they provided an F-Droid repository instead. Doing so
doesn't require them to publish their sources and it's a popular mechanism for
app distribution outside Google Play.

{{</ reactiontext >}}

{{</ sidenote >}}

Token2's tooling tends to look more dated compared to Yubico's (Tkinter GUI
yaaaay). Although the first impression of the Android app is pretty good!

{{< sidenote >}}

{{< reactiontext reaction="smile/27cd" >}}

By the way, FIDO2 management is standardized!

The Token2 app can manage passkeys on a YubiKey just like browsers can manage
passkeys on any FIDO2 key.

{{</ reactiontext >}}

{{</ sidenote >}}

The Token2 PIN+Bio3 key (and other Token2 keys) comes with an accessory
described as a "leather case". That _could_ be relevant for anyone looking to
avoid animal products. However, I emailed Token2 about this and they told me
that the case is in fact made of artificial leather based on polyurethane (a
plastic).

So the keys are probably vegan.

## Conclusion

In terms of features, the PIN+ series lineup by Token2 is likely able to satisfy
my requirements. The Token2 PIN+Bio3 key might additionally help in scenarios
where repeated PIN entry would be annoying, such as SSH authentication using
FIDO2.

I've ordered a Token2 PIN+Bio3. Shipping costs quite a lot from their store so
I've also included a backup key (a cheaper model without biometrics).

If the tiny rats in my brain guiding every decision I make allow it, it I'll
update this post with a diff between expectation and reality.
