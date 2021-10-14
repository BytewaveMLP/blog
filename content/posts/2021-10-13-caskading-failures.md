---
title: "Caskading Failures"
date: 2021-10-13T21:55:15-05:00
draft: true
---

That was yesterday for me.

## First, a little backstory.

So there's this little MLP:FIM community I'm a part of called Cider and Saddle. It's a small bunch, only about 6 people all-told, and we're all good friends. In that group are a couple of nerds like myself who enjoy tinkering with computers and programming. Shortly after I joined CaS, us nerds got together and discussed the idea of building up a set of linked services for the community to use, backed by some kind of internal account system. The specifics of this system will definitely be the subject of a future blog post, but I'll offer some highlights here: at the heart of our system is OpenLDAP and Keycloak, which provide LDAP- and OpenID Connect-based authentication for our services. Then, we have a few different applications up for the community to use, including [Drawpile](https://drawpile.net), and a custom-built file-sharing application we call Cider Cask. All of these are wrapped up in Docker containers, and orchestrated with Compose. Not exactly the cleanest system, but it gets the job done.

Cider Cask is the focus of today's post. It was built by one of our members, who we'll call MF. It's a simple file drop box; users can log in and upload images and other files to the site, and receive short, shareable branded links they can post to our group's chat or share with others. It's a neat little project, and it's been very nice in combination with [ShareX](https://getsharex.com) for sharing screenshots and the like. Cider Cask was built a number of months ago and, since being deployed, hasn't really needed a lot of maintenance. At least, that was true until two weeks ago.

## What went wrong?

In case you hadn't heard, [Let's Encrypt's root certificate expired on September 30th](https://letsencrypt.org/docs/dst-root-ca-x3-expiration-september-2021/), causing many old applications and devices to reject connections to any site secured by certificates issued by Let's Encrypt. At Cider and Saddle, all of our services are backed by a Let's Encrypt wildcard certificate, which we'd configured to automatically renew when needed. We thought that meant we'd be in the clear; after all, we were sure to keep our production system up-to-date, and as long as the system's CA certificates were fresh, there shouldn't be any issues.

We were wrong.

On October 3rd, one of our community members noticed Cask was throwing 500 errors upon visiting the page. Scrubbing through the logs, it was pretty easy to guess what was going on:

```
OpenSSL::SSL::SSLError (SSL_connect returned=1 errno=0 state=error: certificate verify failed (certificate has expired)):
```

We knew from the fact that all of our other services were still online that it wasn't the certificate itself that was the issue, so it was clear that Cider Cask's Docker image must have a stale CAcert file. Therefore, the logical next step would be to rebuild the Docker image with an updated base image, to hopefully get fresh CA certs.

This brought up the first issue: project rot. It had been a while since anyone had touched the Cask codebase, and in that time, a few important developments had occurred; namely, the mimemagic gem was [yanked from Rubygems due to a license violation](https://github.com/mimemagicrb/mimemagic/issues/97). Due to this, we encountered our first hurdle, though it was fairly easily rectified by simply upgrading mimemagic. However, it required MF's presence, as them and myself are the only developers on the CaS tech team with Ruby experience, which added additional delay to brining Cask back online. Once mimemagic was updated, we were able to rebuild the container.

But then we ran into a different problem: `bin/rails` was throwing a `Permission denied` error now. Somehow, permissions had been messed up between the last deployment and this one, and 
