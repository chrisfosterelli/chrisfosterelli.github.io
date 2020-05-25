---
layout: post
permalink: how-contact-tracing-retains-cryptographic-privacy
title: How contact tracing retains cryptographic privacy
image: /image/how-contact-tracing-retains-cryptographic-privacy/social.png
summary: >
  This post breaks down the cryptographic architecture of Apple and Google's new
  contact tracing protocol, and how it preserves your privacy.
---

Apple and Google are working on a new contact tracing protocol, built on top of
Bluetooth broadcasts, that governments can use to build contact tracing mobile
apps for end-users. It just [released this week]. We've heard lots of talk
about how this is "privacy preserving".

You may think that it would be pretty challenging to keep a record of everyone
you've seen, and compare that to a record of who everyone else has seen,
without sharing any records of where anyone has been. This post covers how we
can do it.

<!-- Content Breaker -->

**Quick notes:** If you want the nitty-gritty and are familiar with
cryptography, you'll find the [specification] more appropriate. I skip over
some implementation details, simplify a fair bit, and focus on components that
non-experts may find more valuable for understanding the key ideas. You may
also find this guide a nice companion read to the specification itself.

If you aren't familiar with the concept of Bluetooth-based contact tracing, 
3Blue1Brown has a [high level video describing DP-3T], a protocol which heavily
inspired the Apple / Google project.

## Exposure keys

To start, your phone generates a random key. It's called the *temporary
exposure key*. If you never get sick, this never leaves your phone. You use
this key for one day, after which it is replaced with another random key. Your
phone will keep the past 14 of them.

![A graphic showing the generation of exposure
keys](image/how-contact-tracing-retains-cryptographic-privacy/exposure_keys.svg)

We derive two other keys from the exposure key: one which is used to build the
broadcast identifiers (the *rolling proximity identifier key*) and one which is
used to encrypt metadata sent with those identifiers (the *associated
encryption metadata key*).

We derive these with the *HMAC-based Extract-and-Expand Key Derivation
Function* (HKDF) algorithm. An HKDF is similar to a hash, but instead of a
fixed output we can generate an arbitrary number of bytes.

Additionally, we can apply a *context* to get multiple different outputs from a
single input. We have two keys, so we use two different contexts to get them.
The context is conceptually similar to a hash's salt. In this case our contexts
are the just the strings "EN-RPIK" and "EN-AEMK" -- which are abbreviations for
what they derive. 

![A graphic showing how HKDF derives two
keys](image/how-contact-tracing-retains-cryptographic-privacy/hkdf.svg)

The part which makes this powerful is that HKDF provides us some cryptographic
guarantees: like a hash, we can't easily go backwards to get the exposure key
from either of these output keys, and neither of the derived keys can be used
to easily determine the other key. If we use the same input, we get the same
outputs. In essence, this turns our one random key into two. 

## Proximity identifiers

We then break the day into 10 minute numerical windows. If we start counting at
midnight, 12:00am becomes one, 12:10am becomes two, 12:20am becomes three, and
so on. These values, along with some padding data, are encrypted with AES using
the identifier key to get what we call the *rolling proximity identifiers*.

Your phone broadcasts these identifiers. They will be broadcast along with the
*associated encryption metadata*, which we'll talk about in the next section.

![A graphic that shows how the clock time translates to
identifiers](image/how-contact-tracing-retains-cryptographic-privacy/identifiers.svg)

An interesting note is that AES is not used here with the goal of later
decryption. It's more like HKDF: we want to ensure it can't be undone easily.
The reason we use AES is because, as we will see, deriving the identifiers has
to be done a lot and your phone has hardware acceleration for AES. So the
intention is to make use of AES here to speed derivation up, not to provide
robust encryption and decryption.

We might ask how this relates to physical tracking. Since it changes every 10
minutes, and holds properties that make one identifier unrecoverable from the
other, it retains a robustness to physical tracking. Your phone's Bluetooth MAC
[does this as well]. In fact, the specification deliberately ensures that we
change both the Bluetooth MAC and the proximity identifier at the same time to
avoid correlation. 

## Encryption metadata

In addition to the identifier, your phone will encrypt some metadata with the
metadata key and broadcast that too. We also use AES encryption here; this time
with decryption in mind. The metadata is only decryptable with the day's
exposure key.

If we never share the exposure key, the participants receiving the metadata
will never be able to decrypt it. In theory, the specification supports a
variety of metadata, but the only thing that's included right now is the
Bluetooth power level.

![A graphic that shows how the signal strength is encrypted as
metadata](image/how-contact-tracing-retains-cryptographic-privacy/metadata.svg)

The Bluetooth power level allows participants to calculate an approximation of
how close a given identifier was to them. If we know the power level the device
broadcast with, and the signal strength we received, we can calculate an
approximation of how far the participant with that identifier was from us.

Again, this can only be decrypted if the participant broadcasting it tests
positive and chooses to share their exposure key.

## Broadcasting

So now we've got our two components: the identifier and metadata. These are
what your phone broadcasts over Bluetooth. Every participant near you will see
these broadcasts and record them for a set period of time. Any individual
participant can't correlate any of the individual identifiers and metadata with
any other identifiers and metadata, nor with any Bluetooth MAC address other
than the one it was broadcast with.

![A picture showing a phone broadcasting some identifiers and
metadata](image/how-contact-tracing-retains-cryptographic-privacy/broadcast.svg)

## What happens when a user tests positive

We now have a system to broadcast anonymous identifiers and keep track of them.
How do we know if one of those identifiers is positive, and whether we have
been exposed?

When a participant tests positive, they can upload their recent exposure keys.
Those become *diagnosis keys*: exposure keys for participants who have tested
positive.

An authority will keep a list of all of the diagnosis keys, and your phone will
regularly reach out to download them. For each diagnosis key, we will derive
the full set of identifiers for that day. If any of the generated identifiers
match ones your phone recorded, within a couple hours of tolerance, then we
know you may have been exposed!

Since we have the diagnosis key, we can also derive the associated encryption
metadata key. This is the same key that decrypts the metadata holding the
Bluetooth power level. If you were exposed, we can use it to calculate how
close you may have been.

This touches on why we do all this derivation. Your phone only needs to
download at most 14 diagnosis keys per case, or about 224 kilobytes for 1,000
active cases. If we tried to do this with individual identifiers, we could have
to serve over **32 megabytes** to every phone!

## Conclusion

This is a pretty robust and thoughtful system. It has limitations. For example,
we cannot know precisely where an infected person has been or alert anyone who
isn't participating. 

There are also gaps in the process. How do we stop people who aren't sick from
uploading diagnosis keys as a prank? What is the boundary for being "exposed"?
A few of these issues Apple and Google have left to the app authors. 

Honestly, that feels like the right call. I think Apple and Google did a nice
job of providing expertise in the area they really are experts: cryptographic
systems. The practical components such as communication, testing, vetting, and
so on are left to the medical experts at their respective governments.

I believe what we need next is a reference implementation. There are a lot of
required components that both Apple or Google haven't touched on, and if every
government implements their own system we can expect to end up with some duds.
We need a government to step up, open source their code for implementing this
API, document their process and policies for vetting, and be a repeatable model
for other governments.

But that's not my expertise either. So I'll keep this post to the cryptography.
In that respect, I think this system achieves an artful balance between
protecting user privacy and providing community benefit. For these systems to
work, people have to trust it and want to install it. I consider myself more
privacy conscious than the average person, and I would feel good about using
this on my phone indefinitely.

Thanks to [Caleb Sharp] for reading a draft of this.

[specification]: https://www.apple.com/covid19/contacttracing
[does this as well]: https://www.bluetooth.com/blog/bluetooth-technology-protecting-your-privacy/
[released this week]: https://www.nbcnews.com/tech/tech-news/apple-s-ios-update-here-it-includes-coronavirus-contact-tracing-n1212016
[Caleb Sharp]: https://github.com/calebissharp
[high level video describing DP-3T]: https://www.youtube.com/watch?v=D__UaR5MQao
