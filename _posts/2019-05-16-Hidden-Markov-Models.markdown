---
layout: post
title:  "An Introduction to Hidden Markov Models"
date:   2019-05-16 21:00:00 -0400
categories: design
published: false
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
A Hidden Markov Model, or HMM is a special case of a Bayesian Network.

![image](/assets/images/HMM.png "An image of a basic Hidden Markov Model, 3 hidden states, 3 observations")

In the Bayesian Network representation above of a first-order HMM, there are three hidden states, and three observations.
$$A$$, $$B$$, and $$C$$ form the hidden states, and $$a$$, $$b$$, and $$c$$ form the observations.
What can we observe from the above graph? We can note that the likelihood
of each observation depends solely upon the likelihood of the hidden
state. Namely, the likelihood of $$a$$ is only determined by the likelihood
of $$A$$, and the likelihood of $$b$$ is conditionally independent
from $$A$$ given the likelihood of $$B$$.

# Sources
