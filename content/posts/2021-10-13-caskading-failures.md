---
title: "Caskading Failures"
date: 2021-10-13T21:55:15-05:00
draft: true
---

You ever have one of *those* days? Where everything's falling apart and nothing makes sense? Where you're left wondering "how did any of this ever work?"

Yeah. That was yesterday for me.

## First, a little backstory.

So there's this little MLP:FIM community I'm a part of called Cider and Saddle. It's a small bunch, only about 6 people all-told, and we're all good friends. In that group are a couple of nerds like myself who enjoy tinkering with computers and programming. Shortly after I joined CaS, us nerds got together and discussed the idea of building up a set of linked services for the community to use, backed by some kind of internal account system. The specifics of this system will definitely be the subject of a future blog post, but I'll offer some highlights here: at the heart of our system is OpenLDAP and Keycloak, which provide LDAP- and OpenID Connect-based authentication for our services. Then, we have a few different applications up for the community to use, including [Drawpile](https://drawpile.net), and a custom-built file-sharing application we call Cider Cask. All of these are wrapped up in Docker containers, and orchestrated with Compose. Not exactly the cleanest system, but it gets the job done.

Cider Cask is the focus of today's post. It was built by one of our members, who we'll call MF. It's a simple file drop box; users can log in and upload images and other files to the site, and receive short, shareable branded links they can post to our group's chat or share with others. It's a neat little project, and it's been very nice in combination with [ShareX](https://getsharex.com) for sharing screenshots and the like. Cider Cask was built a number of months ago and, since being deployed, hasn't really needed a lot of maintenance. At least, that was true until two weeks ago.

## What went wrong?

In case you hadn't heard, [Let's Encrypt's root certificate expired on September 30th](https://letsencrypt.org/docs/dst-root-ca-x3-expiration-september-2021/), causing many old applications and devices to reject connections to any site secured by certificates issued by Let's Encrypt. At Cider and Saddle, all of our services are backed by a Let's Encrypt wildcard certificate, which we'd configured to automatically renew when needed. We thought that meant we'd be in the clear; after all, we were sure to keep our production system up-to-date, and as long as the system's CA certificates were fresh, there shouldn't be any issues.

We were wrong.

On October 3rd, one of our community members noticed Cask was throwing 500 errors upon visiting the page. Scrubbing through the logs, it was pretty easy to guess what was going on:

```ruby
OpenSSL::SSL::SSLError (SSL_connect returned=1 errno=0 state=error: certificate verify failed (certificate has expired)):

app/services/openid_service.rb:32:in `request_access_token'
app/controllers/openid_controller.rb:25:in `auth'
```

We knew from the fact that all of our other services were still online that it wasn't the certificate itself that was the issue, so it was clear that Cider Cask's Docker image must have a stale CAcert file. Therefore, the logical next step would be to rebuild the Docker image with an updated base image, to hopefully get fresh CA certs.

This brought up the first issue: project rot. It had been a while since anyone had touched the Cask codebase, and in that time, a few important developments had occurred; namely, the mimemagic gem was [yanked from Rubygems due to a license violation](https://github.com/mimemagicrb/mimemagic/issues/97). Due to this, we encountered our first hurdle, though it was fairly easily rectified by simply upgrading mimemagic. However, it required MF's presence, as them and myself are the only developers on the CaS tech team with Ruby experience, which added additional delay to brining Cask back online. Once mimemagic was updated, we were able to rebuild the container.

## Permissions Hell

When we brought up our stack, we noticed our Cask containers were immediately crashing. Digging into the logs, we found that `bin/rails` was throwing `Permission denied` when trying to run after the container down-leveled its privileges. This is about when I enter the story, as before now I'd been busy with coursework, and had been unable to do any sort of deep investigation. My first instinct was to check executable permissions on the Rails binstub, since that's the obvious answer to `Permission denied`: check the permissions. But that's when I noticed the third major issue:

```shell
root@container:/app/bin# ls -lah /usr/bin/env
-rwxr-xr-x 1 root root 48K Sep 24  2020 /usr/bin/env
root@container:/app/bin# su cidercask
bash-5.1$ /usr/bin/env # the shebang for bin/rails
bash: /usr/bin/env: Permission denied
bash-5.1$ ls -la bin
ls: error while loading shared libraries: libpcre2-8.so.0: cannot open shared object file: Permission denied
```

*wat.*

That wasn't supposed to happen. That couldn't happen. Right? Surely nothing in our `Dockerfile` should do anything to break something as critical as **Coreutils** of all things. And yet, after looking at the official Debian `slim` image, the official Ruby 2.6 `slim` image, and [`ruby-node`](https://hub.docker.com/r/timbru31/ruby-node/)'s `2.6-slim` tag, I was unable to find any sort of break in the chain that would cause something like this. So if the images themselves were fine, it had to be something we were doing wrong. But... what?

It was at this point that another friend I was chatting with posed an interesting point: what about `/usr`? Specifically, did I make sure that my user had search permissions for `/usr/bin`? It hadn't occurred to me that that might be an issue, since... well, what's going to change permissions of `/usr` from anything other than `755`? And yet...

```shell
root@container:/app/bin# ls -la /usr
total 44
drwxrws--- 1 root root 4096 Aug 16  2020 .
drwxr-xr-x 1 root root 4096 Oct 13 03:54 ..
drwxrws--- 1 root root 4096 Aug 16  2020 bin
...
```

`750`? And is that... a sticky bit? Where the hell did that come from? Well...

```shell
bw81@production-host:~srv/cider-cask$ ls -la docker
...
drwxrws---+  3 cas srv 4.0K Aug 15  2020 usr/
...
```

For reference, files in `docker/` were used as part of our *development* Docker builds for running every component of the application inside one container using `s6`. They had no real use in production, but we used one `Dockerfile` for both builds (perhaps to our detriment, but that's a topic for another time).

So we'd found our culprit. For some reason, long ago when this server was first deployed, long before we'd even started working on the Cider and Saddle infrastructure project, these permissions had been set up this way. `750`, with the sticky bit sit, and some extended ACLs that I don't believe served any purpose. Docker had apparently copied these strange permissions over to our container image when it was built this time, despite never doing so before. Now, this was fine for most of the project files, since they were `chown`ed to the `cidercask` user. But `docker/usr`... we hadn't thought of that. So Docker "helpfully" copied those permissions for us, which left `/usr` readable only by `root:root`. Which meant no searching for binaries and shared objects for any user that wasn't `root`.

Great. There went 4 hours of my life I'll never get back. So we reset the permissions for the `srv` user's homedir to make sure nothing like this would happen again, and rebuilt the container for what would hopefully be the last time. And with that, we brought up the stack one last time, confirmed everything was working, and all got to bed on time that night.

... I'm just kidding.

```ruby
OpenSSL::SSL::SSLError (SSL_connect returned=1 errno=0 state=error: certificate verify failed (certificate has expired)):

app/services/openid_service.rb:32:in `request_access_token'
app/controllers/openid_controller.rb:25:in `auth'
```

Goddammit.

## Square One

So all of that was for nothing. We rebuilt the container on a completely fresh base image, one that's used by thousands of projects, and yet somehow... somehow we were still running into the Let's Encrypt issue.

Okay, so... let's check our CA certificates. If the image maintainers won't update them for some reason, I guess we can always update them ourselves.

```shell
root@container:/app# apt-get update && apt-get install ca-certificates
...
ca-certificates is already the newest version (20210119).
```

What?! Those are fresh! Like, two-days-old fresh!

Maybe it was an outdated version of OpenSSL? I know Debian tends to ship with fairly old versions of packages, but I would be surprised if they skimped out on keeping OpenSSL fresh.

```
root@container:/app# openssl version
OpenSSL 1.1.1k  25 Mar 2021
```

That's pretty recent. Certainly not the latest, but certainly newer than the [1.1.0 version Let's Encrypt warned everyone to upgrade to](https://community.letsencrypt.org/t/openssl-client-compatibility-changes-for-let-s-encrypt-certificates/143816).

Were we somehow just... missing the ISRG root CA?

```shell
root@container:/app# ls -lah /etc/ssl/certs | grep -i isrg
lrwxrwxrwx 1 root root   16 Sep 28 21:32 4042bcee.0 -> ISRG_Root_X1.pem
lrwxrwxrwx 1 root root   51 Sep 28 21:32 ISRG_Root_X1.pem -> /usr/share/ca-certificates/mozilla/ISRG_Root_X1.crt
```

No luck. Maybe having the old, expired DST root cert around was making OpenSSL unhappy? This would violate everything I know about how certificate chain validation works, but... [it seems to have worked for this StackOverflow user](https://stackoverflow.com/a/69407814). But not even removing the DST X3 cert and rebuilding `cacert.pem` fixed it.

At this point, it was getting pretty late. Dealing with this whole mess had been pretty tiring, and it was getting hard to think straight. After jokingly suggesting to MF that we just disable SSL verification in Cask as a quick patch, I decided to put things on hold for a bit and quietly cry myself to sleep that night.

## Refreshed Perspective

The moment I woke up, it hit me. The OS had fresh CA certs, that much was true. But Rails is an old framework, and I know that, back in the day, we couldn't always rely on the OS for up-to-date CA certs out-of-the-box. This led a lot of library/application maintainers to ship their own CA bundles that would hopefully be more reliable or, at the very least, more predictably located. So I started digging through code, trying to find where any HTTP calls where made, and what library they were using. I started with `request_access_token`, the method which was `raise`ing the error in the first place. That led me to the `openid_connect` gem which, after a small amount of its own logic, wound up in `rack-oauth2`, in a method which was responsible for completing the OAuth2 flow by exchanging the given authorization code from the user's browser with the authenticating server for an access token. To complete this HTTP transaction, `rack-oauth2` made use of [the `httpclient` gem](https://rubygems.org/gems/httpclient), which was last updated *5 years ago*. Digging into [its GitHub repository](https://github.com/nahi/httpclient), I quickly found the root cause of all these issues: [a `cacert.pem` bundle](https://github.com/nahi/httpclient/blob/4658227a46f7caa633ef8036f073bbd1f0a955a2/dist_key/cacerts.pem) that was last updated **11 YEARS AGO**!

11 years is a long time. In that time, a lot has happened in the world of certificate authorities, like [the untrusting of Symmantec's entire PKI](https://security.googleblog.com/2018/03/distrust-of-symantec-pki-immediate.html) and, y'know, **the founding of Let's Encrypt**. Seriously, this CA bundle was from a time where Let's Encrypt didn't even exist! What a dark era of network security that was. When this certificate bundle was last updated, I was 10 years old, and had just finished the 4th grade.

No wonder modern infrastructure gets breached all the time.

Anyway. It turns out, the maintainers of `rack-oauth2` had actually taken notice to this stale CA cert bundle, and released a version of `rack-oauth2` to work around this issue by having `httpclient` use the system's CA bundle instead of its own. Guess when that fix was committed and released?

[13 days ago](https://github.com/nov/rack-oauth2/commit/4a207ed785cecc15e8b20f34f9acd6094634fa30). On the same day that the LE root cert expired. I mean... I'm glad somebody finally noticed. But the fac that this 11-year-old cert bundle was—and might still be!—buried inside potentially thousands of production applications for who-knows-how-long actually scares me. Yikes.

Anyway. On that note, we quickly upgraded `rack-oauth2`, rebuilt the container, and brought up the stack for the last time. And with that, all was well. We had successfully restored Cider Cask after only... 7 days of partial outage, and 6 days of total outage. Not the worst, but certainly suboptimal.

## Closing Thoughts

It's times like these that I remember why I both love nad hate what I do. Working in software engineering/devops means you'll inevitably be faced with those kinds of impossible problems, the ones that should never happen, could never happen, and yet are totally happening, and it's your job to fix them. In times like those, it's important to question everything; don't leave any assumptions unturned, and don't just check the usual suspects. When your infrastructure seems to be falling apart, it's important to assume it's already come down, and audit everything. Never assume "it can't be X, I know it isn't X." Because it will be X. It's always X. Sometimes even literally.

But I also love these kinds of situations. Not being knee deep in total production collapse or anything, mind you; that's never fun. But there's something immensely satisfying about *figuring it out*, about finding the needle in the haystack, about wading neck-deep into the source code nad coming out with the answer. And the deeper you go, the more exciting it is to finally be done with it.

I'd also like to mention once again just how scary it is that `httpclient`'s CA bundle has been this stale for this long. The latest release has over **2 million** downloads on RubyGems. How many of those dependents are aware of the stale CA bundle? How many know the magic commands to get it to wipe its CA store and load the OS' defaults instead? If Let's Encrypt's root CA hadn't expired recently, would anyone have noticed?

I leave you with two important lessons: when you're faced with that impossible problem, *check everything*, no matter how silly; and for the love of God, **keep tabs on your dependencies**, and don't let project rot turn a stale codebase into a vulnerability.
