---
layout: post
title:  "Dependency Injection in C++"
date:   2018-09-06 21:00:00 -0400
categories: design
---
<!-- You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets: -->

This website is named after Dependency Injection because it was the first
design pattern I was introduced to in my career. 

The usefulness of
Dependency Injection is in that the implementation of a class
is more flexible and independent by decreasing the amount of coupling
between a client and another service. The exact decoupling is the removal
of a client class's dependency on the implementation of an interface, instead
requiring the implementation to be passed in.

Let's take a look at a class that requires a dependency and the class
defining
that dependency.

```cpp
#include <iostream>

class GasolineSource{
public:
    void FuelUp() = 0;
};

class GasStation : public GasolineSource{
public:
    void FuelUp(){
        std::cout << "Pumping gas at gas station" << std::endl;
    }
};

// Car is dependent on GasStation being defined in order to fuel up.
class Car{
    GasStation gasolineService;
public:
    Car(){ }
    void getGasoline(){
        std::cout << "Car needs more gasoline!" << std::endl;
        gasolineService.FuelUp();
    }
};
```

Here, the class _Car_ is dependent on the definition for _GasStation_.
_Car_ cannot be defined without _GasStation_ being defined and _Car_
would need to be changed whenever the source of gasoline is changed,
like to a _can_.

```cpp
#include <iostream>

class GasolineSource{
public:
    void FuelUp() = 0;
};

class FuelCan : public GasolineSource{
public:
    void FuelUp(){
        std::cout << "Pumping gas from fuel can" << std::endl;
    }
};

// Car is dependent on FuelCan being defined in order to fuel up.
class Car{
    FuelCan gasolineService;
public:
    Car(){ }
    void getGasoline(){
        std::cout << "Car needs more gasoline!" << std::endl;
        gasolineService.FuelUp();
    }
};
```

The solution for this issue is Dependency Injection.

```cpp
#include <iostream>

class GasolineSource{
public:
    virtual void FuelUp() = 0;
    virtual ~GasolineSource() = default;
};

class GasStation : public GasolineSource{
public:
    virtual void FuelUp(){
        std::cout << "Pumping gas at gas station" << std::endl;
    }
};

class FuelCan : public GasolineSource{
public:
    virtual void FuelUp(){
        std::cout << "Pumping gas from fuel can" << std::endl;
    }
};

class Car{
    GasolineSource *gasolineService = nullptr;
public:
    // The dependency for a source of gasoline is passed in
    // through constructor injection
    // as opposed to hard-coded into the class definition.
    Car(GasolineSource *service)
    : gasolineService(service) {
        // If the dependency was not defined, throw an exception.
        if(gasolineService == nullptr){
            throw std::invalid_argument("service must not be null");
        }
    }
    void getGasoline(){
        std::cout << "Car needs more gasoline!" << std::endl;
        // Abstract away the dependency implementation with polymorphism.
        gasolineService->FuelUp();
    }
};
```

Here is some code in main showing how the dependencies were injected.

```cpp
int main(){
    GasolineSource *stationService = new GasStation();
    GasolineSource *canService = new FuelCan();

    // racecar is independent from the implementation of the fuel service.
    // a gas station service is injected.
    Car racecar(stationService);
    racecar.getGasoline();

    // dune buggy is independent from the implementation of the fuel service.
    // a fuel can service is injected.
    Car duneBuggy(canService);
    duneBuggy.getGasoline();

    delete stationService;
    delete canService;
    return 0;
}
```

Here is the output from running main, showing that the appropriate services
were called based up the dependencies injected.

{% highlight bash %}
$ ./main
Car needs more gasoline!
Pumping gas at gas station
Car needs more gasoline!
Pumping gas from fuel can
{% endhighlight %}

Please feel free to email me with any additional questions or concerns at
[{{ site.email }}](mailto:{{ site.email }}).
