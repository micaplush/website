---
title: Potentially interesting stuff in my Nix-based homelab
date: 2025-04-05T00:00:00Z
---

This article is the first in a series about stuff I've developed for my homelab.
I'm mostly writing this as a braindump for my future self but it might be useful
to others as well. Feel free to use ideas and code from this as you see fit.

This post in particular will just be a high-level overview of what we're talking
about and some cherry-picked highlights of what makes it interesting. Oh and at
the end I'll touch on the status quo of the project.

{{< sidenote >}}

Side note: When I say braindump I mean it. I hope it'll be somewhat
comprehensible what's going on but I've never been awfully good at writing I
think. If you have any feedback, feel free to contact me.

Other side note: This kind of box is for **side notes**. It can be freely
ignored while skimming but it might have some useful context or witty
commentary.

{{</ sidenote >}}

## Why should you care?

Basically I've developed some very weird and very experimental stuff that I
thought oughta exist and then I've integrated nearly everything into a monorepo.
(And I haven't "un-integrated" anything for this series.) Here are some of the
most interesting components of that repo.

### Agenix secrets generation

Secrets (when using agenix or a similar tool) are normally opaque blobs that you
commit into your repository and hopefully document how to recreate. I've
completely automated that. (Almost) all of my secrets have a declarative
specification attached that describes how they're generated.

```nix
{
  x.global.agenix.secrets = {
    "radicale/htpasswd" = {
        generation.template = {
            data.users = [ "alice" "bob" ];
            content = ''
                {{- range .users -}}
                {{ . }}:{{ hashBcrypt (readSecret (fmt "radicale/passwords/%s" .)) 10 }}
                {{ end -}}
            '';
        };
    };
  };
}
```

My generator heavily relies on declarative templates instead of scripts to
generate secrets. That makes it possible to **deterministically replay
templating** for a secret given a stream of bytes to use whenever randomness is
required.

In practice that means secrets generation is a single idempotent command that
will finish with all secrets up to date and up to spec. I don't have to think
about secrets anymore and they're perfectly documented.

Oh and JSON can be templated using plain Nix data structures because generating
structured data with text templates is _✨ awful ✨_.

### Globalmods

Essentially "globalmods" are Nix modules whose options live in a global
"namespace". I.e. values defined on one host are automagically visible on all
hosts.

For example I can just write:

```nix
{
  x.global.email.accounts = {
      "paperless@systems.tbx.at" = {
          permissions.receive = true;
      };
  };
}
```

Which is essentially saying "to whom it may concern, I want an email account".
Then I can consume that definition in the module that sets up my email server,
even if that's on a different host than where the email account was requested.

This is _really neat_ since it allows me to collocate logically related config
in a single module instead of moving it out into a central file where all email
accounts are defined (for example).

### Static "service discovery"

I hate relying on copy/pasting URLs to connect different services in my infra.

Therefore I've written a globalmod (vertical integration, yeah! yeah!) that
provides a central place where modules can say "this host provides a network
service called bar on these ports called http and https". And then other
modules, possibly on other hosts, can say:

```nix
{
  services.foo.peerURL = "http://${globalConfig.netsrv.services.bar.fqdn}";
}
```

Some cool benefits you gain for free with something like that:

- You can move services between hosts without fear.
- Stuff instantly explodes in your face if you reference a service that doesn't
  exist.
- You can automate firewall rules and DNS zone files with this.
  - And remember, globalmods span across hosts, so this works even for services
    not running on the same server as your DNS.
- You can generate beautiful internal landing pages like this one:

![A website listing out a bunch of internal service names and ports. The page is adorned by trans pride flags.](./dir.png)

This idea is kinda sorta a little bit stolen from Kubernetes' `Service`
resource.

## Other stuff

Those are approximately the biggest flashiest features I've implemented now.
Each of them is gonna get a dedicated writeup where I go in depth about how it
works and how it's implemented. But there's also a couple other things that
might be worth documenting:

- A flake-parts module for writing [Justfiles](https://github.com/casey/just) in
  Nix
- Generally my flake-parts setup because that wasn't super straightforward to
  learn
- Some cool shenanigans for building Go packages while referencing libraries
  locally in the monorepo
- Some of the services making use of globalmods (OIDC, Email, Prometheus scrape
  targets, Restic backup repositories, ...)
- Other services that I've added weird hacks to

Also I might write about ideas that I've been thinking about but haven't
implemented. This would probably be about deployment and release automation.

## Current status and what's next

I'm currently working on publishing a sanitized version of my monorepo (removing
encrypted secrets just to be extra-sure as well as some files that can't be
included for copyright or other reasons). The structure of that repo will be
explained across subsequent posts to keep the amount of information in each post
bite-sized.

Truthfully, there haven't been any big developments in this project over the
last few months. This is in part due to personal reasons (life just had
unexpected things for me in store) but also &mdash; and I'm sorry to bring this
up again &mdash; the community shakeups that occurred in 2024 kinda limited my
enthusiasm for Nix.

To be clear, I think it's good that systemic problems in the community and
governance structures were exposed and that committed people tried to fix them.
What's disappointing to me is the _nuclear response_ this caused, leading to the
eventual resignation of several highly respected community members.

Maybe I shouldn't complain about this. I've mostly been a passive user of Nix so
far and I haven't done anything to support what I think were the voices to move
Nix forward while all of this was happening. I'm still including this section
because ultimately I want Nix to have a better, more inclusive public image and
not talking about it doesn't get us anywhere. And I can't be the only one who
thinks this way. {{< small >}}(...Right?){{< /small >}}

It should also be noted that the vast majority of people in the Nix community
are absolutely fine and it would be unfair of me to approach anyone with
prejudice. If you're interested in anything I've described in this post, I'd be
thrilled to hear from you! I don't have concrete plans to split anything out of
my monorepo and turn it into its own community project but if there's interest
in that, we can definitely do it. Just shoot me a message using any of the
contact methods below. 😊
