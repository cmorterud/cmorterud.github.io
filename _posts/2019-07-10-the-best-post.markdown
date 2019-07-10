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

![image](/assets/images/HMM-three.png "An image of a basic Hidden Markov Model, 3 hidden states, 3 observations")

In the Bayesian Network representation above of a first-order HMM, there are three hidden states, and three observations.
$$A$$, $$B$$, and $$C$$ form the hidden events, and $$a$$, $$b$$, and $$c$$ form the observations.
The above HMM is first-order because the likelihood of each
hidden event (except the first), is determined solely
by the likelihood of the previous event. The above HMM would be second
order if there was a dependency, or arrow pointing from $$A$$ to $$C$$,
as $$C$$ is dependent upon $$A$$ and $$B$$ in that case, 
as opposed to solely $$B$$ in the above graph.

The likelihood for the above graph
is simple to compute because of the simple dependency structure.
$$P(A,B,C,a,b,c)=P(A)P(a|A)P(B|A)P(b|B)p(C|B)p(c|C)$$.

The events $$A$$, $$B$$, and $$C$$ are used to represent latent
variables. The events $$a$$, $$b$$, and $$c$$ are used
to represent observations.

## Modelling

A HMM can be used to model a sequence of events with observations.
A common situation is that someone goes to the doctor,
and indicates their symptoms. The doctor either determines
the patient is sick or healthy, based upon the reported symptoms.
This someone goes to the doctor three separate occasions,
and this is modelled by the below 1st order HMM.

![image](/assets/images/HMM-doctor.png "An image of a Hidden Markov Model, modelling a patient with a Doctor")

The above 1st order HMM could be a model for a patient
seeing a doctor on three separate occasions. On the first
and second visit, the patient was coughing. On the last
visit, the patient was smiling. Whether
the patient was sick or healthy is unobserved, but a probabilistic
argument can be made. We can generally guess
that a patient is sick if they are coughing, and healthy if they are not,
but it is possible a patient could be coughing when they see a doctor,
and not be sick. A patient could not feel well or be sick, but still be smiling and happy.

Let's explicitly define the possible events for each random variable.
For each hidden variable, Day One State, Day Two State, and Day Three State,
the patient is either sick or healthy.
For each observation, the patient is either coughing or smiling.

Let's define the probabilitiy of Day One State.

| State | Likelihood |
|---------|-------|
| Healthy | 0.80 |
|---------|-------|
| Sick |  0.20 |

where the first entry indicates the probability that
the patient is healthy, 80%, and the second entry
indicates the probability that the patient is sick, 20%.

Let's define the transition probabilities,
namely the likelihood of the next hidden state
given the current hidden state.

|  | Healthy | Sick |
|:-------:|---------|-------|
| Healthy |  0.90 |  0.10 |
| Sick | 0.50 | 0.50 |

The rows indicate the current state, and the columns indicate
the next state. For example, the probability 
of becoming sick if the patient is healthy is 10%.

Let's define the emission probabilities,
namely the probability that coughing or smiling
is observed from the patient.

|  | Smiling | Coughing |
|:-------:|---------|-------|
| Healthy |  0.75 |  0.25 |
| Sick | 0.40 | 0.60 |

For example, the probability that the patient would be coughing
if he or she is sick is 60%.

With the probabilities defined, we can compute likelihoods
for the model.


# Sources
