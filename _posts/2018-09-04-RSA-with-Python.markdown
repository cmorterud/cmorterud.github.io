---
layout: post
title:  "RSA Encryption and Decryption example by Python"
date:   2018-09-04 21:00:00 -0400
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

## Walkthrough
Let's check that we are using the same version of Python
{% highlight bash %}
$ python3 --version
Python 3.6.5
{% endhighlight %}

To begin, RSA requires two distinct prime numbers, commonly known
as $$p$$ and $$q$$.

For our example, let $$p=19$$ and $$q=41$$. Both of these
values are private. I picked those at random.

Next, let $$n=pq=779$$. $$n$$ is used as a modulus in the RSA cryptosystem.

Next, we need to compute Euler's totient function for $$n$$, 
which is $$\lambda(n)$$. Euler's totient function is defined for
an integer $$x$$ as the count of numbers less than $$x$$ that are relatively
prime to $$x$$, which in layman's term means the amount of integers
less than $$x$$ that share no factors with $$x$$. For a prime number $$p$$,
$$\lambda(p)=p-1$$, because a prime number has no factors besides 1 and $$p$$.

Thus, $$\lambda(n)=\lambda(pq)=\lambda(p)\lambda(q)=18\times40=720$$.
This is possible because Euler's totient function has
the property of multiplicativity.  This is a private
value.

Next, we need to find an integer $$e$$ such that $$1<e<\lambda(n)$$ and 
the greatest common denominator of the totient of $$n$$ and $$e$$ is 1, or
$$gcd(e, \lambda(n))=1$$. For this example, I choose 7.

Lastly, we need to find an integer $$d$$ such that
$$d\equiv e^{-1}\mod \lambda(n)$$, which means that $$d$$ is the 
modular multiplicative inverse of $$e$$ modulo $$\lambda(n)$$. This 
is the first part where the math gets tricky, but I can give an algorithm
written in Python that will find the value $$d$$ given $$e$$
and $$\lambda(n)$$.

{% highlight python %}
# Extended Euclidean Algorithm
def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    g, y, x = egcd(b % a, a)
    return (g, x - (b // a) * y, y)

# application of Extended Euclidean Algorithm to find a modular inverse
def modinv(a, m):
    g, x, y = egcd(a, m)
    if g != 1:
        raise Exception('modular inverse does not exist')
    return x % m
{% endhighlight %}

{% highlight bash %}
$ python3
>>> modinv(7, 720)
103
{% endhighlight %}
Thus $$e=7$$ and $$d=103$$. Let's check that 103 is the modular multiplicative
inverse of 7 modulo 720, which means that $$7\times103 \mod 720 = 1$$.
{% highlight bash %}
$ python3
# pow is a built-in python3 function that raises the first
# argument to the second modulo the third argument
>>> pow(7 * 103, 1, 720)
1
{% endhighlight %}

# Application
To encrypt a message $$M$$, compute $$M^e\mod n=C$$,
where C is the ciphertext. Let our example message
be 5.

It's important to keep in mind that $$M$$ must be coprime to $$n$$
for a multiplicative modular inverse to exist (the inverse is the "encryption"
of $$M$$).
The factors of $$n$$ are $$p$$ and $$q$$, thus as long as $$M$$ is less than
$$n$$ and not equal to $$p$$, $$M$$ can be encrypted.

{% highlight bash %}
$ python3
>>> pow(5, 7, 779)
225
{% endhighlight %}

Thus, our ciphertext is 225.

To decrypt an encrypted message $$C$$, compute $$C^d\mod n$$.

{% highlight bash %}
$ python3
>>> pow(225, 103, 779)
5
{% endhighlight %}
Thus, we have successfully encrypted and decrypted an message.

The message of 5 isn't very useful, but for example, one could convert 
ASCII characters into integers, and then individually encrypt each character.
That would probably take too long, so an optimization would be encrypting
a number that represents for example 4 characters, or 8, or 16, or so on,
up unto the maximum integer that can be encrypted, which is 1 less than
the lesser of $$p$$ and $$q$$.


# Sources
[Detailed rundown and proofs of correctness](https://sites.math.washington.edu/~morrow/336_09/papers/Yevgeny.pdf)

[Source for Extended Euclidean Algorithm](https://en.wikibooks.org/wiki/Algorithm_Implementation/Mathematics/Extended_Euclidean_algorithm)


