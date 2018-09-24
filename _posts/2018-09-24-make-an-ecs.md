---
author: Razakhel
title: Make your own ECS (Entity-Component System)
featimg: ECS_shapes.jpg
tags: [C++, ECS, Game engine, Architecture]
category: [Tutorial]
---

"ECS" is an acronym you can read quite a lot, since it seems to have been the common hype about game engines for the past few years. This tutorial/article is meant to get you to understand its concepts, and get a grasp of a basic implementation to build your own.

#### What **IS** an ECS?

The acronym **ECS** stands for **E**ntity-**C**omponent **S**ystem. It is an architecture pattern mostly applied to game engines

Long story short: there is no proper definition of what an ECS actually is and, as such, there is no single way to implement it.

An "ECS" clearly isn't something well-defined. From what I've read about it though, I'd define the usual ECS being an implementation with:

- The Component pattern: prefering aggregation of characteristics over inheritance (_`has a`_ associations instead of _`is a`_ ones);
- The Data locality pattern: optimizing CPU cache by conserving data (here our entities) in a single contiguous collection;
- The Update method pattern: make base classes for systems & components with a virtual `update` method specialized for each inherited class.

###### Disclaimer

You may have read/heard some statements that led you here, such as:

*- ECS is good for performance*

It depends. Implementing only the Component pattern won't give you a performance boost. There's a good chance it will be the opposite, actually. But it will give you an insane amount of flexibility, which is not negligible.

*- An ECS is way more memory efficient than an OOP-based approach*

Well, this is not wrong... but not entirely true either. Removing the inheritance avoids having a pointer to every inherited object due to the virtualization. On the other hand, in our ECS some space is lost because of the way of manipulating components & systems by indices. I lack experience to say this for sure, but I guess that in the long run it indeed tends to be way more memory efficient than the OOP-based design.

*- Every major game engine uses an ECS, I should make one for my own*

True. Although actually, Unity's current state [isn't truly an ECS](https://answers.unity.com/questions/669643/entity-component-system.html?childToView=1076793#answer-1076793), since the systems are not accessible and thus not modular. It however implements the Component pattern. As a matter of fact, they're in the process of changing the whole architecture into a real ECS at the moment.

That being said, converting your own OOP-based engine into an ECS should not be a brainless decision. If your current engine is simple enough and well architectured, you could lose a fair amount of time while gaining absolutely no performance at all, even losing some. But sure enough, you will gain a ton of flexibility, which should be your main reason when evolving into one.

---

Bottom line is: don't exert yourself trying to implement something you may not have any gain from if you have no good reason to do so. ***However***, if your goal is to learn, then I can't hold you back from implementing one. Break everything you want in your current architecture, start from scratch, do whatever you want with it, but I think learning should never be prevented in any way.

#### Component

This part is actually what an ECS is really about. I guess a lot of people use the acronym "ECS" to talk about this specific pattern.

A _really_ good reference for this is [the article from Game Programming Patterns](http://gameprogrammingpatterns.com/component.html).

#### Data locality

Small explanation: a computer's CPU possesses a cache, whose size vary but still is pretty small (for example, L3 cache (largest but slowest) is ~6 MB on Intel i5s, ~8 MB on i7s, ~19 MB on AMD Ryzen 5s and ~20 MB on Ryzen 7s. This list is absolutely not entirely accurate, it may be different with each generation & model). As you can see, this cache is incredibly smaller than your amount of RAM, but is _really_ faster than it.

Nowadays, CPUs perform what we can call _caching_: copying data from the RAM into the cache so that it can access it **way** faster. However, this operation only copies data adjacent to the one you're accessing. If the data you're fetching from memory isn't in this cache slice, this causes a "cache miss" which slows down the program to pick data from the RAM. That is, your program can be executed faster if you're often selecting contiguous data.

_There is no way that I can explain better than [the Game Programming Patterns' article on this](http://gameprogrammingpatterns.com/data-locality.html), so I'll let you read it if you didn't understand._

In that way, entities are meant to be in a single memory-contiguous collection (like an std::vector in C++, std::Vec in Rust, etc). As such, CPU caching doing its job and your entire world being contained in this collection, you minimize the amount of cache misses and allow your program to process data faster.

#### Update method

This one's the easiest: in our System base class we will define a virtual methode `update`, which will be reimplemented by the inheriting systems. This will be where all our logic is, executed for each iteration of the game loop.

#### References

- [What's an Entity System? - Wikidot](http://entity-systems.wikidot.com/)
- [Example of ECS implementation: EntityX - Alec Thomas](https://github.com/alecthomas/entityx)
- [Example of ECS implementation: anax - Miguel Martin](https://github.com/miguelmartin75/anax)
- [Component-based engine design - Randy Gaul](http://www.randygaul.net/2013/05/20/component-based-engine-design/)
- [Sane usage of Components and Entity systems - Randy Gaul](http://www.randygaul.net/2014/06/10/sane-usage-of-components-and-entity-systems/)
- [Writing a game engine in 2017 - Randy Gaul](http://www.randygaul.net/2017/02/24/writing-a-game-engine-in-2017/)
- [Implementation of a component-based entity system in modern C++ - Vittorio Romeo, CppCon 2015](https://www.youtube.com/watch?v=NTWSeQtHZ9M)
- [Game entity management basics - Vittorio Romeo, Dive into C++11](https://www.youtube.com/watch?v=QAmtgvwHInM)
- [Component pattern - Game Programming Patterns](http://gameprogrammingpatterns.com/component.html)
- [Data Locality pattern - Game Programming Patterns](http://gameprogrammingpatterns.com/data-locality.html)
- [Update Method pattern - Game Programming Patterns](http://gameprogrammingpatterns.com/update-method.html)
