---
layout: post
title:  "Genetic Algorithm in C#"
date:   2018-10-1 10:00:00 -0400
categories: development
published: false
---
<!-- You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets: -->

Genetic Algorithms are used to solve optimization problems where there
exists a function to evaluate the fitness of a particular potential solution.

For example, finding the value of a random bitstring given
a fitness function to determine the similarity between a bitstring
and the solution.

## Fitness

In genetics terminology, a potential solution is referred to as a chromosome.

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

The function Fitness returns a score in the range of (0, 1] indicating
the relative fitness of the chromosome.

## Algorithm

There are four steps in running a Genetic Algorithm, namely
Selection, Crossover, Mutation, and Repeated Iterations.

# Selection
Given some sample population, which in 
our example is a population of bitstrings, we should select
two bitstrings according to fitness proportionate selection, where
a bitstring is more likely to be chosen if the fitness of that bitstring
is higher.

This means for our target bitstring of `0001`, `0010` is more likely
to be selected in our Selection step than `1000`.

Here is an example code that runs a Selection step.

{% highlight cs %}
public string Select(IEnumerable<string> population,
                     IEnumerable<double> fitnesses,
                     double sum = 0.0)
{
    // fitness proportionate selection.
    var fitArr = fitnesses.ToArray();
    if(sum == 0.0){
        foreach(var fit in fitnesses){
            sum += fit;
        }
    }
    
    // normalize.
    for(int i = 0; i < fitArr.Length; ++i){
        fitArr[i] /= sum;
    }
    
    var popArr = population.ToArray();
    
    // sort fitness array along with population array.
    Array.Sort(fitArr, popArr);
    
    sum = 0.0;
    
    // calculate accumulated normalized fitness values.
    var accumFitness = new double[fitArr.Length];    
    for(int i = 0; i < accumFitness.Length; ++i){
        sum += fitArr[i];
        accumFitness[i] = sum;
    }
    
    var val = random.NextDouble();
    
    for(int i = 0; i < accumFitness.Length; ++i){
        if(accumFitness[i] > val){
            return popArr[i];
        }
    }
    return "";
}
{% endhighlight %}

# Crossover
The two chromosomes from the Selection step should now
be crossed over with some probability (~0.60).
If a crossover occurs, the crossover will happen at a random
position.

For example, with two chromosomes `1110` and `1001` and
randomly generated position 2, the chromosomes become
`1010` and `1101`.

{% highlight cs %}
public IEnumerable<string> Crossover(string chromosome1,
                                     string chromosome2)
{
    int randomPosition = random.Next(0, chromosome1.Length);
    string newChromosome1 = chromosome1.Substring(randomPosition) + chromosome2.Substring(0, randomPosition);
    string newChromosome2 = chromosome2.Substring(randomPosition) + chromosome1.Substring(0, randomPosition);
    return new string[] { newChromosome1, newChromosome2 };
}
{% endhighlight %}


# Mutation

# Repeated Iterations
The above three steps should be applied repeatedly until
a new population is generated, as opposed
to the first iteration, where a sample population is randomly
generated. This new population
will become the population to select new chromosomes from.

Please feel free to email me with any additional questions or concerns at
[{{ site.email }}](mailto:{{ site.email }}).
