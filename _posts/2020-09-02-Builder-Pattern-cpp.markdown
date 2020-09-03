---
layout: post
title:  "Builder Pattern in C++"
date:   2020-09-02 21:00:00 -0400
categories: design
---

## Introduction
The builder pattern is the usage of a public static inner class to facilitate
construction of objects with many member variables, especially of the same type.
The key advantages
of using the builder pattern is readability as well as extensibility
when using inheritance.


## An Example
```cpp
#include <string>

class Automobile {
private:
    int numberOfTires;
    std::string bodyType;
    std::string engineType;
    int fuelTankSizeInGallons;
    bool airConditioned;
    int odometerMiles;

public:
    Automobile(int numberOfTires, std::string bodyType, std::string engineType,
            int fuelTankSizeInGallons, bool airConditioned, int odometerMiles)
            : numberOfTires(numberOfTires), bodyType(bodyType), engineType(engineType),
            fuelTankSizeInGallons(fuelTankSizeInGallons), airConditioned(airConditioned),
            odometerMiles(odometerMiles) {}
};

int main() {
    Automobile automobile(4, "coupe", "V8", 16, true, 20000);
}
```
This class has several member variables. The specific constructor arguments
may be a little obvious based upon the values placed, or if there are variables,
even moreso obvious. This obviousness may be more lacking for a different class.
We can make the parameters more obvious by introducing an inner static class.
`static` because this inner class will not depend on any specific
instances of `Automobile`.

## Telescoping Constructor Anti-Pattern
What if we wanted to make `airConditioned` optional? That would require
modification of any constructor that uses `airConditioned`. If a
class uses the telescoping constructors anti-pattern, this could require
a partial rewrite of the constructors.
```cpp
class Automobile {
private:
    int numberOfTires;
    std::string bodyType;
    std::string engineType;
    int fuelTankSizeInGallons;
    bool airConditioned;
    int odometerMiles;

public:

    class Builder {
        int numberOfTires;
        std::string bodyType;
        std::string engineType;
        int fuelTankSizeInGallons;
        bool airConditioned;
        int odometerMiles;
    }

    Automobile(int numberOfTires, std::string bodyType, std::string engineType,
            int fuelTankSizeInGallons, bool airConditioned, int odometerMiles)
            : numberOfTires(numberOfTires), bodyType(bodyType), engineType(engineType),
            fuelTankSizeInGallons(fuelTankSizeInGallons), airConditioned(airConditioned),
            odometerMiles(odometerMiles) {}
    // DO NOT DO THIS
    Automobile(int numberOfTires, std::string bodyType, std::string engineType,
            int fuelTankSizeInGallons, bool airConditioned)
            : Automobile(numberOfTires, bodyType, engineType,
            fuelTankSizeInGallons, airConditioned, 0)
    // DO NOT DO THIS
    Automobile(int numberOfTires, std::string bodyType, std::string engineType,
            int fuelTankSizeInGallons)
            : Automobile(numberOfTires, bodyType, engineType,
            fuelTankSizeInGallons, false)
    // DO NOT DO THIS
    Automobile(int numberOfTires, std::string bodyType, std::string engineType)
            : Automobile(numberOfTires, bodyType, engineType, 0)
    // DO NOT DO THIS
    Automobile(int numberOfTires, std::string bodyType, std::string engineType)
            : Automobile(numberOfTires, bodyType, engineType, 0)
    // DO NOT DO THIS
    Automobile(int numberOfTires, std::string bodyType)
            : Automobile(numberOfTires, bodyType, "")
    // DO NOT DO THIS
    Automobile(int numberOfTires)
            : Automobile(numberOfTires, "");
    // DO NOT DO THIS
    Automobile()
            : Automobile(0);
};
```
Notice how each constructor has an additional parameter, and each constructor
is a dependency of the next? This is known as telescoping constructors.
Any change in variables other than `odometerMiles` will require a rewrite.
Please do not use telescoping constructors.

## Builder Pattern
```cpp
#include <string>

class Automobile {
private:
    int numberOfTires;
    std::string bodyType;
    std::string engineType;
    int fuelTankSizeInGallons;
    bool airConditioned;
    int odometerMiles;

public:

    class Builder {
    public:        
        int numTires = 0;
        std::string body = "";
        std::string engine = "";
        int fuelTankSize = 0;
        bool airCondition = false;
        int odometer = 0;

        Builder* numberOfTires(int numberOfTires) {
            this->numTires = numberOfTires;
            return this;
        }

        Builder* bodyType(std::string bodyType) {
            this->body = bodyType;
            return this;
        }

        Builder* engineType(std::string engineType) {
            this->engine = engineType;
            return this;
        }

        Builder* fuelTankSizeInGallons(int fuelTankSizeInGallons) {
            this->fuelTankSize = fuelTankSizeInGallons;
            return this;
        }

        Builder* airConditioned(bool airConditioned) {
            this->airCondition = airConditioned;
            return this;
        }

        Builder* odometerMiles(int odometerMiles) {
            this->odometer = odometerMiles;
            return this;
        }

        Automobile build() {
            return Automobile(*this);
        }
    };

    Automobile(Builder builder)
            : numberOfTires(builder.numTires), bodyType(builder.body), engineType(builder.engine),
            fuelTankSizeInGallons(builder.fuelTankSize), airConditioned(builder.airCondition),
            odometerMiles(builder.odometer) {}
};

int main() {
    Automobile automobile = Automobile::Builder().numberOfTires(4)
        ->bodyType("coupe")
        ->engineType("V8")
        ->fuelTankSizeInGallons(16)
        ->airConditioned(true)
        ->odometerMiles(20000)->build();
}
```
The parameters are clearly identified by the member functions of the builders,
and optional parameters are easily implemented (simply remove the function call!).

## Thanks!
Please feel free to email me at [{{ site.email }}](mailto:{{ site.email }})
with any questions or concerns. 
Thanks for reading!
