---
layout: post
title:  "Factory Pattern in C++"
date:   2018-09-12 21:00:00 -0400
categories: design
---
<!-- You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets: -->

The factory pattern is part of a class of design patterns called
creational design patterns, and
the point of the factory pattern is to remove the explicit instantiation
of a class out of a user's code to reduce dependencies.

Let's talk about some code in a library. Let's say I had some classes.
{% highlight cpp %}
#include <iostream>

class Animal{
public:
    virtual void MakeNoise() = 0;
    virtual ~Animal(){}
};

class Dog : public Animal {
public:
    virtual void MakeNoise(){
        std::cout << "Woof woof!" << std::endl;
    }
};

class Cat : public Animal {
public:
    virtual void MakeNoise(){
        std::cout << "Meow!" << std::endl;
    }
};
{% endhighlight %}
So there is a `Dog` class, which inherits from `Animal`, and a `Cat` class,
which also inherits from `Animal`.
Let's take a look at some client code using these classes.
{% highlight cpp %}
#include <iostream>

int main(){
    int type = 0;

    Animal *animalPtr = nullptr;

    std::cin >> type;

    if(type == 0){
        animalPtr = new Cat();
    }
    else if(type == 1){
        animalPtr = new Dog();
    }

    animalPtr->MakeNoise();

    return 0;
}
{% endhighlight %}

So in this client's code, the client reads in an integer, and then uses
that integer to determine what dynamic type `animalPtr` should be.

What if I decide to add a new animal to my animal kingdom?

{% highlight cpp %}
class Bird : public Animal{
    virtual void MakeNoise(){
        std::cout << "Chirp chirp!" << std::endl;
    }
};
{% endhighlight %}
Now, the client's code must be recompiled to account for the new animal
that can be instantiated. The client must add a new
`else if` chain to account for the new Bird animal. 
As in, the client's code depends on my
class definitions for my animals.

The solution to this problem is known as the factory pattern.
The factory pattern provides a method to instantiate other classes,
thus decoupling the responsibility for instantiation from a client's code.

<script src="https://gist.github.com/cmorterud/04f33ff8eb1bcfba0138f0e945671d7b.js"></script>

Of course, in this example, everything is in the same file, but if `main`
and the library code (`Cat`, `Dog`, `Animal`, `AnimalFactory`)
were in separate files, a new class definition derived from Animal
would not require a code change on the client side, because
the responsibility for instantiation is on `AnimalFactory`, which
is part of the library code.

Please feel free to email me with any additional questions or concerns at
[{{ site.email }}](mailto:{{ site.email }}).
