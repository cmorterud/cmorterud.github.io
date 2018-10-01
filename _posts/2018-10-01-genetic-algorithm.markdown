---
layout: post
title:  "Genetic Algorithm in C#"
date:   2018-10-1 9:00:00 -0400
categories: development
---
<!-- You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets: -->

Genetic Algorithms are used to solve optimization problems where there
exists a function to evaluate the fitness of a particular potential solution.

For example, finding the value of a random bitstring given
a fitness function to determine the similarity between a bitstring
and the solution.

In genetics terminology, a potential solution is referred to as a chromosome

To use a Genetic Algorithm, we need a fitness function which
is passed a bitstring, and returns a score that represents the fitness
of the bitstring.

{% highlight csharp %}
public class FitnessHelper{
    // assumes chromosome and solution are bitstrings
    private string solution;
    public double Fitness(string chromosome){
        int editDistance = Compute(chromosome, solution);
        double score = 1.0 / (double) editDistance;

        return score;
    }

    public FitnessHelper(string targetSolution){
        solution = targetSolution;
    }

    // https://en.wikipedia.org/wiki/Levenshtein_distance
    // https://www.dotnetperls.com/levenshtein
    // dynamic programming
    private static int Compute(string s, string t)
    {
        int n = s.Length;
        int m = t.Length;
        int[,] d = new int[n + 1, m + 1];

        // Step 1
        if (n == 0)
        {
            return m;
        }

        if (m == 0)
        {
            return n;
        }

        // Step 2
        for (int i = 0; i <= n; ++i){
            d[i, 0] = i;
        }
        for (int j = 0; j <= m; ++j){
            d[j, 0] = j;
        }

        // Step 3
        for (int i = 1; i <= n; i++)
        {
            //Step 4
            for (int j = 1; j <= m; j++)
            {
                // Step 5
                int cost = (t[j - 1] == s[i - 1]) ? 0 : 1;

                // Step 6
                d[i, j] = Math.Min(
                    Math.Min(d[i - 1, j] + 1, d[i, j - 1] + 1),
                    d[i - 1, j - 1] + cost
                );
            }
        }
        // Step 7
        return d[n, m];
    }
}
{% endhighlight %}

Please feel free to email me with any additional questions or concerns at
[{{ site.email }}](mailto:{{ site.email }}).
