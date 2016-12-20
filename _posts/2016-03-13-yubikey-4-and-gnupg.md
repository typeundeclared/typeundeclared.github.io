---
layout: post
title: Setting up the Yubikey 4, GnuPG Issues
tags: [misc]
---
# Yubikey 4 and GnuPG

I recently decided to purchase two Yubikeys, version 4.  The intent here was to:

- Take advantage of Chrome and U2F, because getting a call/text to your phone on every sign-on gets a little annoying.
- Have something more convenient/safe to put PGP keys on (a smart card/key takes the keys off your main computer and makes them harder to access).

There are other ways to use the Yubikeys, such as OTP through the Yubi servers, but I am not currently using any services that hook into those (yet).

## Gmail, Chome and U2F

This really did as out of the box as advertised.  No complaints.

## Setting up OTP

Even though I had no use cases for it, I decided to set up OTP mode anyways.  Turns out that my Linux distro (Mint 17.3) had packages for the yubikey-personalization-gui and the associated library, libykpers-1-1.  Great.  Installed, run... and it tells me that there is no Yubikey inserted (there was one).

Hrmm.  After great searching through the forums, it turns out that libykpers-1-1 is version 1.14 in Mint, and the Yubikey 4 needs 1.17.  Adding Ubuntu xenial to my apt sources got my the appropriate version.  After that, the manual for the Yubico Personalization GUI worked out fine.

*I did try to build it and then install it somewhere on my system that wouldn't get in conflicts with the package manager, and while I could have made it work, messing with the system lib paths is a little uncomfortable, so I decided on the Ubuntu source repos instead.*

## GnuPG and the Yubikey 4

One of the reasons I opted for a Yubikey 4, rather than a cheaper U2F device and a separate openPGP smart card was that supposedly, it would fit both bills.  Several years ago, I generated a 4096 bit key, which I've only ever used to transfer documents between family members (who had their own key).  4096... was probably overkill.  I didn't really know that at the time, and I had just chosen the biggest keysize available at the time.

Fast forward, I have a new(ish) Linux box, and two Yubikeys.  I've started wanted to backup documents online, but I don't entirely trust Google, so I'd like to encrypt them (to myself) for online storage.

### Backups

Before I did anything, I backed up the master key (both a tar copy and a straight cp -r copy) to USB sticks.

Then, I made a .gnupg directory copy for each of the two Yubikeys, since the keytocard is somewhat destructive (removes the private key entirely).  Then both of these directories would be likewise backed up.  Ultimately, I'll only have one .gnupg directory, the one matching my daily-use Yubikey, but I'll have backups for the other Yubikey, as well as the master backup.  *Maybe this is the wrong way to approach this?  Thoughts?*

So what's the first problem I run into?  Trying to perform a 'keytocard' tells me that the Yubikey only support 2048 bit keys... which is *not* what the documentation says.  Hrmm.

Perhaps I need GnuPG2?  That supposedly has updates designed for more modern smart cards.  Same issue (2048 bits max).  What is going on?

Well, GnuPG has actually two 2.x versions: stable (2.0) and modern (2.1), modern being akin to an unstable/testing.  Apparently modern will support cards with keysize 4096+, but isn't generally available in ubuntu or mint.  So, once again, I go in search of an apt repo that has the right versions, which leads me to debian unstable.  *I hope this doesn't bite me (too hard)*.  For reference, see: [Yubikey 4 GPG Key Size forum post](http://forum.yubico.com/viewtopic.php?f=35&t=2205).

Also, *do not have an out of date version of scdaemon*.  This was quite confusing, because moving keys to the card worked fine, but I could not decrypt anything existing:
    gpg: public key decryption failed: Wrong secret key used
    gpg: decryption failed: No secret key
You'll get a message like:
    gpg: WARNING: server 'scdaemon' is older than us