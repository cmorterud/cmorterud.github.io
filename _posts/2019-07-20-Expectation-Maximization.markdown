---
layout: post
title:  "An Introduction to Expectation Maximization with a Binomial Distribution"
date:   2019-07-20 21:00:00 -0400
categories: design
published: false
---
<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/latest.js?config=TeX-MML-AM_CHTML"></script>
<!-- You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets: -->
## Introduction
Consider an unfair coin, with some probability $$p$$ of heads,
and $$1-p$$ probability of tails. A process to identify
the parameter of the underlying distribution, or identify
the probability of heads is Expectation Maximization.

Expectation Maximization is the process of maximizing
the likelihood of the observed data with respect
to the chosen model by tuning the parameter of the model.
A simple, intuitive example of this is 
identifying the bias of an unfair coin.

<script src="https://gist.github.com/cmorterud/084e3298581fc1e91128916bd5a9af03.js"></script>

![image](/assets/images/20_choose_15_likelihood_graph.png "A graph showing the likelihood of 20 choose 15 with respect to a binomial model")

The graph shows the likelihood of $$f(p)={20\choose15}(p)^{15}(1-p)^5$$ for $$[0.0, 1.0]$$
maximizing at 0.75. Thus, 0.75 maximizes the likelihood of the observations.
This is an intuitive result, yet demonstrates the idea of likelihood maximization.

## Modeling


## Likelihoods


### Derivation

### Application


# Sources
[N choose K code](https://stackoverflow.com/questions/4941753/is-there-a-math-ncr-function-in-python)

# Contact
Please feel free to email me at
[{{ site.email }}](mailto:{{ site.email }})
with any questions or concerns. Thanks!