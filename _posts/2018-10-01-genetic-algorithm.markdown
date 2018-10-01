---
layout: post
title:  "Genetic Algorithm Example in C#"
date:   2018-10-1 10:00:00 -0400
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

The purpose of the Selection step is to generate chromosomes,
with a preference towards chromosomes with a higher fitness score.

The Crossover and Mutation steps introduce randomization
to the generated chromosomes.

Repeated Iterations eliminates the noise from the random steps (ideally!)
to identify a solution with maximum fitness.

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

Randomness is also introduced through mutations,
where each position of each chromosome has a chance to be modified,
where in this example means a bit flip.

{% highlight cs %}
public string Mutate(string chromosome, double probability)
{
    string  ret = "";
    double randomVariable = 0.0;
    foreach(char c in chromosome){
        randomVariable = random.NextDouble();
        if(randomVariable < probability){
            if(c == '1'){
                ret += "0";
            }
            else{
                ret += "1";
            }
        }
        else{
            ret += c;
        }
    }
    return ret;
}
{% endhighlight %}

# Repeated Iterations
The above three steps should be applied repeatedly until
a new population is generated, as opposed
to the first iteration, where a sample population is randomly
generated. This new population
will become the population to select new chromosomes from.

A new population is generated many times to help eliminate random noise
from the population.

The parameters of the function that actually runs the genetic algorithm
are a function mapping bitstring to a fitness score,
the length of the bitstrings being generated, the probability
of a crossover, the probability of a mutation per position per bitstring,
and the number of iterations to run.

{% highlight cs %}
public string Run(Func<string, double> fitness, int length, double crossoverProb, double mutationProb, int iterations = 100)
{   
    int populationSize = 500;
    // run population is population being generated.
    // test population is the population from which samples are taken.
    List<string> testPopulation = new List<string>(), runPopulation = new List<string>();
    string one = "", two = "";
    var randDouble = 0.0;
    
    // construct initial population.
    while(testPopulation.Count < populationSize){
        testPopulation.Add(Generate(length));
    }

    var fitnesses = new double[testPopulation.Count];
    
    double sum = 0.0;
    
    // continuously generate populations until number of iterations is met.
    for(int iter = 0; iter < iterations; ++iter){
        runPopulation = new List<string>();
        
        // calculate fitness for test population.
        sum = 0.0;
        fitnesses = new double[testPopulation.Count];
        for(int i = 0; i < fitnesses.Length; ++i){
            fitnesses[i] = fitness(testPopulation[i]);
            sum += fitnesses[i];
        }
        
        // a population doesn't need to be generated for last iteration.
        // (using test population)
        if(iter == iterations - 1) break;
        
        while(runPopulation.Count < testPopulation.Count){
            
            one = Select(testPopulation, fitnesses, sum);
            two = Select(testPopulation, fitnesses, sum);

            // determine if crossover occurs.
            randDouble = random.NextDouble();
            if(randDouble <= crossoverProb){
                var stringArr = Crossover(one, two).ToList();
                one = stringArr[0];
                two = stringArr[1];
            }

            one = Mutate(one, mutationProb);
            two = Mutate(two, mutationProb);

            runPopulation.Add(one);
            runPopulation.Add(two);
        }

        testPopulation = runPopulation;
    }
    
    // find best-fitting string.
    var testSort = testPopulation.ToArray();
    var fitSort = fitnesses.ToArray();
    
    Array.Sort(fitSort, testSort);
    
    return testSort[testSort.Length - 1];
}
{% endhighlight %}

At the end, the best fitting bitstring from the last population
is returned. 

Increasing the amount of iterations helps to remove more and more noise
from the populations, but also increases the runtime of the algorithm.
A balance needs to be found between accuracy and runtime requirements.

Increasing the population size increases the amount of randomness
in the algorithm, which is needed to eventually ideally
generate the best fitting bitstring. Population size and number
of iterations should increase and decrease proportionally,
because as additional randomness is introduced, additional iterations
are needed to filter out the randomness to identify the most fit solution.

## Applications
This example finds a randomly generated bitstring given a fitness function.
That application may not be useful for most users, but problems
can be adapted to use a Genetic Algorithm.

For example, given a problem to identify which numbers
in a given array sum to a certain value, a bitstring can represent
the numbers included in a potential solution.

Given the list `[1, 2, 3, 4]` and a sum of `8`, the bitstring
`1011` represents a solution where first, third and fourth element
of the array are summed which is a solution to the problem.

Creating a function to represent the fitness of a bitstring is fundamental
to using a Genetic Algorithm. One could write a function
that, given a bitstring, calculates the sum, and returns the
difference between the bitstring represented sum and the target sum,
representing the fitness of the bitstring.

## Full Implementation
<script src="https://gist.github.com/cmorterud/0a77492852e0a5ecc4d80b361329656e.js"></script>

## Contact
Please feel free to email me with any additional questions or concerns at
[{{ site.email }}](mailto:{{ site.email }}).

## Sources

[Genetic Algorithm Wiki](https://en.wikipedia.org/wiki/Genetic_algorithm)

[Less technical article](https://towardsdatascience.com/introduction-to-genetic-algorithms-including-example-code-e396e98d8bf3)
