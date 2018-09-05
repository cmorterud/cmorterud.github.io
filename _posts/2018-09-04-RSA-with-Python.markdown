---
layout: post
title:  "RSA Encryption and Decryption example by Python"
date:   2018-09-04 15:00:00 -0400
categories: development
---
<!-- You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets: -->
<!-- 
I have been a longtime macOS user since I started college, staying
as far away from Windows as I could. That was mainly due to my college's use
of Red Hat Enterprise Linux and macOS's command line utilities and Homebrew
out of the box being
a near approximation for RHEL's utilities.  -->

RSA stands for Rivest, Shamir, and Adleman. The most common usage
of RSA is the cryptosystem, one of the first asymmetric 
cryptosystem. By asymmetric, I mean that the key to encrypt and
the key to decrypt are different, as opposed to a system like the
Advanced Encryption Standard, where the key used to encrypt and decrypt
are exactly the same.

What this means is that a person named Alice can generate a pair of RSA keys
to communicate securely,
where the key used to encrypt known as the public key, and the key used
to decrypt known as the private key. Alice can then distribute
their public key, and then other people such as Bob can use that key
and the RSA encryption algorithm to send secure messages to Alice.
Alice can also sign messages by encrypting with her private key,
and then Bob can use the known public key to decrypt the message,
meaning that the given message could only have come from Alice.

Together, Alice and Bob can each generate a RSA keypair,
and use those to communicate securely. 

When Gnu Privacy Guard 
otherwise known as GPG uses your RSA private key to sign
a message, and someone else's RSA public key to encrypt a message,
GPG is just performing RSA encryption with your private key, and then 
RSA encryption with someone else's RSA public key.

To begin, RSA requires two distinct prime numbers, commonly known
as $$p$$ and $$q$$.






