---
layout: post
title:  "Diffie Hellman Example with Python"
date:   2018-12-22 21:00:00 -0400
categories: development
---
<script type="text/x-mathjax-config">
    MathJax.Hub.Config({
        tex2jax: {
          skipTags: ['script', 'noscript', 'style', 'textarea', 'pre']
        }
      });

    MathJax.Hub.Queue(function() {
        var all = MathJax.Hub.getAllJax(), i;
        for(i=0; i < all.length; i += 1) {
            all[i].SourceElement().parentNode.className += ' has-jax';
        }
    });

  </script>

<script type="text/javascript" src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
<!-- You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets: -->
## Introduction
Diffie Hellman key exchange is a method of constructing a shared secret
between two parties. A shared secret is something that can be used as a 
key to conduct symmetric encryption, such as AES. The Diffie Hellman
key exchange depends on the difficulty of the discrete log problem,
which is generally believed to be hard in this current day and age.
Understanding this usually requires some understanding of mathematical
groups.

## Discrete Log Problem
This discrete log problem is the issue of finding the exponent
needed to raise some number $$b$$ to yield some other number $$a$$,
within some group $$G$$. For this post, we will assume the group
to be the integers modulus $$p$$, or $$\mathbb{Z}_p$$, and the group operation
to be multiplication modulo $$p$$. $$p$$ should be a prime number
to ensure the generator $$g$$ cycles over the entire domain
of $$\mathbb{Z}_p$$.

This is finding $$k$$ in the following equation,
$$a=b^k$$, where $$a$$ and $$b$$ are given, and $$\mod p$$ is implied.

## Diffie Hellman Key Exchange
Let there be a public generator $$g=2$$ of some group $$G=\mathbb{Z}_{41}$$, with associated
$$p=41$$.

{% highlight python %}
>>> g = 2
>>> p = 41
>>> a = 6
>>> pow(g,a,p)
23
{% endhighlight %}
Alice generates some secret value $$a=6$$, computes $$g^a=23$$,
and sends this to Bob. Notice because the discrete log problem
is hard ($$a$$ and $$p$$ are sufficiently large to prevent a brute force attack),
some eavesdropper is unable to find $$a$$ given $$g^a$$.

{% highlight python %}
>>> g = 2
>>> p = 41
>>> b = 5
>>> pow(g,a,p)
32
{% endhighlight %}
Bob does the same for secret value $$b=5$$, sending $$g^b=32$$ to Alice.
Notice an eavesdropper cannot compute $$b$$ from the public value $$g^b$$.

Alice and Bob can then generate shared secret $$g^{ab}$$.
{% highlight python %}
# Alice computing g^(ab)
>>> g_b = 32
>>> a = 6
>>> pow(g_b, a, p)
40
{% endhighlight %}

{% highlight python %}
# Bob computing g^(ab)
>>> g_a = 23
>>> b = 5
>>> pow(g_a, b, p)
40
{% endhighlight %}
$$g^{ab}=40$$, which is now a shared secret. This shared
secret could be used as a key, or as a seed for a 
pseudo random number generator to generate many keys for transmitting
data with a stream cipher.

# Sources
[Great video on the Diffie Hellman key exchange](https://www.youtube.com/watch?v=NmM9HA2MQGI)