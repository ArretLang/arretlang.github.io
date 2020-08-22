---
layout: default
title: Arret Types
nav_order: 2
description: "Arret type system overview"
permalink: /types
last_modified_date: 2020-08-22
---

# Arret types

Arret uses a powerful, but human-centric type system. Types in Arret are tied to what they represent, how they are stored or handled 
by a computer is secondary. Types can be annotated with constraints that can be checked at compile-time. One of the most powerful features 
of Arret's type system is the ability to safely cast between types using *shortest-path conversion*.

## Simple Type Example

Let's imagine you are writing a service that needs to handle temperatures from a variety of sensors around the world, configured based on their local scale.

Start by importing the type definitions for `DegreesC` and `DegreesF` from the Arret community:

```
from community.arretlang.org import {
  DegreesC, DegreesF
}
```

These types already have a viable conversion path in the type definition, so you can cast between them seamlessly:
```
DegreesC euro_temp := 20:
DegreesF us_temp := (DegreesF) euro_temp; # us_temp is now 68
```

How did this work? Behind the scenes, types in the community have a number of `cast_to` functions that allow for easy translation of values. As an example, 
let's now add a new type for Kelvin, the SI temperature scale that starts at absolute zero.

```
type {simple, public} Kelvin {
  description: "Used to represent a temperature value in Kelvins"
  aliases: K
  values: {
    Float val
  }
  init: Kelvin(Float k) { val := k }
  constraints: {
    requires val >= 0
  }
}
```

This type allows for the easy use of Kelvin, but the type system cannot cast to other units:
```
Kelvin k := 1000
Kelvin k2 := -20 # Compile-time error: Kelvin requires val >= 0
DegreesC euro_temp := 20:
Kelvin k_temp := (Kelvin) euro_temp; # Compile-time error: No path between DegreesC and Kelvin
```

We now define a `cast_to` function to allow for conversion between Kelvin and DegreesC:
```
def {pure, invertible} cast_to(Kelvin k, DegreesC c) {
  c := k - 273.15
}
```

By marking the `cast_to` as invertible, Arret now can use this cast function for converting from Kelvin to DegreesC, and its inverse function (adding 273.15) 
to convert back to Kelvin from DegreesC. Since Arret's types represent human-centric values, the type system can automatically build transitive casts. The 
type system represents types as nodes in a graph, and `cast_to`s as edges, and performs a shortest-path search to compose a casting function. Like in the simple
case, if a conversion path cannot be found, it will report a compile-time error:
```
DegreesC euro_temp := 20:
DegreesF us_temp := (DegreesF) euro_temp; # us_temp is now 68
Kelvin k_temp := (Kelvin) us_temp; # k_temp is now 293.15
```

## Compound Type Example

Types can also be more complex, consisting of e.g., a ratio of two typed values, like miles per hour, or gallons per minute. The shortest path search can handle 
these compond types in the same manner, allowing for greater reuse of `cast_to`s for the simple types. Continuing our example from above, let's supposed we would 
like to use the sensor data in a time series and report the changes of temperature over an interval. Arret has built-in support for ratio types, and converting 
between the three dimensions (e.g., m, m^2, m^3):

```
from community.arretlang.org import {
  DegreesC, Hour, Minute
}

# Assume the sensor interval is an hour
Hours interval := 1;

# First temp reading
DegreesC t1 := 20;
# Second temp reading (1 hour later)
DegreesC t2 := 26;

DegreesC/Hour change := (t2 - t1) / interval # change is now 6 DegreesC/Hour
```
