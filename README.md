# How to Generate a Collection of Random Numbers in Modern C++

#### Author: Jonathan Boccara
#### Published May 24, 2019
#### See: [Fluent{C++}](https://www.fluentcpp.com/)
#### @see: <https://www.fluentcpp.com/2019/05/24/how-to-fill-a-cpp-collection-with-random-values/>
#### @see: https://stackoverflow.com/questions/15155778/superscript-in-markdown-github-flavored
#### @see: https://gist.github.com/molomby/9bc092e4a125f529ae362de7e46e8176

Filling out a collection with random numbers is C++ is an easy thing to conceive, but it isn’t that easy to guess how to implement.

In this article you will find the following:

- how to generate a random number in modern C++ (it’s not with `rand()` any more),
- how to override the contents of an existing collection with random numbers,
- how to generate a new collection filled with random numbers.

## Generating random numbers in modern C++

To generate random numbers with C++, we need to be able to generate random numbers on a computer in the first place. But this is contradictory: a computer is a **deterministic** machine!

### Generating random numbers with a deterministic machine

Solving that contradiction is not as philosophical as it looks: the random numbers generated by the C++ standard library, like most random numbers in program, are **not random**. But they look random enough to fit the purposes of most programs that need numbers drawn at random, and for that reason they are called “pseudo-random”.

How does this work? In some simple random generators, every time you ask for a random number you get the next element of a sequence of numbers (*X&#x2099*) whose definition looks like this:

> *Xn+1 = (A.X&#x2099 + B) mod C*

And A and B and C are large numbers carefully chosen so that the generated numbers (the Xn) are evenly distributed, to look like random numbers. Some statistical tests, such as the [chi-square test](https://en.wikipedia.org/wiki/Chi-squared_test), allow to evaluate how evenly a sequence of numbers is distributed, how random it looks.

This is called a linear congruential generator and is amongst the simplest formulae for random number generators. Although the C++ standard library offers such a generator, it also offer others, such as the [Mersenne Twister](https://en.wikipedia.org/wiki/Mersenne_Twister) generator, which use more elaborate formulae and are more commonly used.

Such random numbers engine need to be initialized: every time we need a number, we get the **next** element of a sequence, but how does the sequence gets its first element? It can’t be hardcoded, otherwise you would always get the same sequence of random numbers for every run of the program. And this wouldn’t look random at all.

So we need another component, in charge of igniting the random engine with an initial value. This component can draw that value from a current state in the hardware, or can itself have a pseudo-random engine. But the point of the matter is that it can generate a number that is not always the same between two runs of the program.

Finally, the raw numbers generated by the random engine may not have the distribution you want: perhaps you want numbers evenly distributed between 1 and 6, or numbers that follow a normal distribution.

For that we need a third component, the distribution, to channel the output of the random engine into a certain distribution.

In summary, we need 3 components:

- a random device to ignite the random engine,
- the random engine that runs the formulae,
- the distribution.

### The <random> features of modern C++

Before C++11, the standard way to generate random numbers was to use `rand()`. But rand() didn’t have a generation (nor a design) of very high quality, so the standard C++ library got new components to generate random numbers in C++11.

The design of those components follow the model we’ve seen:

- The random generator to initiate the random engine is called `std::random_device`,
- There are several random engines, a common one being Mersenne Twister with default parameters implemented in `std::mt19937`,
- And there are several distributions, for instance the `std::normal_distribution` for Normal law, or `std::uniform_int_distribution` for randomly distributed integers between two boundaries.

### Code example

Let’s now put all this into code:

```
std::random_device random_device;
std::mt19937 random_engine(random_device());
std::uniform_int_distribution<int> distribution_1_100(1, 100);

auto const randomNumber = distribution_1_100(random_engine);

std::cout << randomNumber << '\n';
```

Note how the random device produces an initial value when called on its `operator()`. To generate a random number, we then only need the distribution and the initiated engine.

Also note that none of the three components participating to the generation can be const, as all those operations modify their internal states.

Now let’s run that code. It outputs:

```
54
```

How random does that look?

### Filling a collection with random numbers

Now that we know how to generate one random number, let’s see how to fill a collection with random numbers. Let’s start with how to override the contents of an existing collection, and move on to how to generate a new collection with random numbers.

One way to go about that could be to loop over the contents of the collection, invoke the above random number generation, and write the results in the collection:

```
std::random_device random_device;
std::mt19937 random_engine(random_device());
std::uniform_int_distribution<int> distribution_1_100(1, 100);

std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

for (size_t i = 0; i < numbers.size(); ++i)
{
    numbers[i] = distribution_1_100(random_engine);
}
```

But this code shows a lot of technical details:

- all the components of random number generations,
- the internals of a for loop.

All those low-level details lying around get in the way of reading code, all the more so it’s the middle of other operations on the collection.

Let’s replace this with a call to a standard STL algorithm: `std::generate`. `std::generate` takes a range and a function that can be called with no arguments, and fills the range with the values returned by that function.

Sounds not too far from what we have here. We only need to generate a function that returns random values generated by our three components. Let’s start by writing the desired calling code:

```
std::generate(begin(numbers), end(numbers), RandomNumberBetween(1, 100));
```

Or even better, let’s hide the iterators taken by the standard algorithm, with a version taking a range:

```
ranges::generate(numbers, RandomNumberBetween(1, 100));
```

Here is a possible implementation for that ranges version of the algorithm:

```
namespace ranges
{
    template<typename Range, typename Generator>
    void generate(Range& range, Generator generator)
    {
        return std::generate(begin(range), end(range), generator);
    }
}
```

Now how do we implement the function object `RandomNumberBetween`? We need to pass the two boundaries in its constructor, and its `operator()` must return a random number.

Note that there is no need to create a new random engine for each random draw, so we can store the engine and distribution in the function object:

```
class RandomNumberBetween
{
public:
    RandomNumberBetween(int low, int high)
    : random_engine_{std::random_device{}()}
    , distribution_{low, high}
    {
    }
    int operator()()
    {
        return distribution_(random_engine_);
    }
private:
    std::mt19937 random_engine_;
    std::uniform_int_distribution<int> distribution_;
};
```

In C++14, generalized lambda capture allows us to implement this with a lambda (thanks Avinash):

```
auto randomNumberBetween = [](int low, int high)
{
    auto randomFunc = [distribution_ = std::uniform_int_distribution<int>(low, high), 
                       random_engine_ = std::mt19937{ std::random_device{}() }]() mutable
    {
        return distribution_(random_engine_);
    };
    return randomFunc;
};
```

Let’s now run the calling code:

```
std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
ranges::generate(numbers, RandomNumberBetween(1, 100));
```

And see what’s inside the collection:

```
for (int number : numbers)
{
    std::cout << number << ' ';
}
```

When I ran the code, it output:

```
58 14 31 96 80 36 81 98 1 9
```

## Generating a collection with random numbers

Now that we know how to fill an existing collection with random numbers, adding new elements to a collection is only one algorithm away: we use `std::generate_n` instead of `std::generate`.

`std::generate_n` does the same thing as `std::generate`, but with a different interface: instead of taking a begin and an end iterator, `std::generate_n` takes a begin and a size N. It then does a generation starting from the begin and going on for N times.

There is a trick associated to this interface: passing an output iterator such as `std::back_inserter` as a “begin” iterator. The effect is that the algorithm will write N times to this iterator, that will itself `push_back` N values to a container.

Here is what it looks like in code:

```
std::vector<int> numbers;
std::generate_n(std::back_inserter(numbers), 500, RandomNumberBetween(1, 100));

for (int number : numbers)
{
    std::cout << number << ' ';
}
```

Here is the output of this program:

```
86 35 65 3 90 78 63 87 49 62 94 84 56 32 69 41 99 47 95 28 15 7 99 47 3 62 10 66
35 49 83 85 76 82 79 66 44 42 16 17 1 62 74 9 11 42 74 50 72 25 4 81 10 16 98 33
64 24 6 90 16 72 93 61 86 48 57 25 61 18 7 20 50 68 80 38 87 70 20 81 58 29 99 81 
25 49 59 14 15 98 68 32 46 1 99 74 56 21 27 52 22 67 86 81 25 50 14 82 56 10 8 16 
87 63 40 6 64 56 3 31 95 12 16 5 20 15 42 90 21 69 87 86 37 58 60 11 13 38 66 70 
40 36 49 25 57 73 77 19 39 48 61 19 47 14 11 31 70 39 78 33 100 2 24 54 76 94 69 
63 63 49 79 6 21 62 24 83 70 50 7 33 98 78 48 93 65 48 98 70 15 57 4 10 82 30 39 
90 32 45 80 21 53 98 5 71 92 25 30 92 45 19 13 1 55 51 15 25 4 98 77 37 55 56 92 
70 74 49 1 25 64 80 14 76 66 94 46 15 59 26 66 3 17 44 40 8 49 50 43 32 99 17 81 
48 30 6 68 48 66 32 27 26 19 58 27 71 36 7 70 78 35 1 32 48 37 12 70 30 84 37 14 
72 46 28 87 94 11 19 53 20 20 28 63 49 68 42 34 47 100 94 65 44 97 53 67 57 73 78 
67 15 42 90 7 25 93 5 29 11 50 85 51 49 84 41 94 8 21 1 71 15 5 86 42 74 20 64 44 
52 35 38 89 45 69 36 54 57 65 1 60 34 66 10 4 38 90 35 66 32 61 49 15 82 36 68 54 
72 24 30 59 34 23 84 68 65 68 36 32 11 14 9 49 95 84 29 16 52 84 36 23 6 18 38 45 
76 26 37 35 17 43 17 46 58 10 46 22 31 28 27 69 66 62 91 19 91 26 25 84 48 31 62 
86 87 50 56 98 58 20 24 29 50 6 18 11 64 6 63 69 47 97 7 39 61 47 100 49 33 45 70 
68 21 79 19 21 1 69 28 75 22 91 9 2 47 87 34 16 78 3 96 92 92 29 15 98 20 48 95
73 98 86 48 62 48 18 68 23 54 59 6 80 88 36 88 33 58 10 15 17 55 79 40 44 56 
```

Oh, this is so random.

Here is all the code put together:

```
#include <algorithm>
#include <iostream>
#include <random>
#include <vector>

namespace ranges
{
    template<typename Range, typename Generator>
    void generate(Range& range, Generator generator)
    {
        return std::generate(begin(range), end(range), generator);
    }
}

// C++11
class RandomNumberBetween
{
public:
    RandomNumberBetween(int low, int high)
    : random_engine_{std::random_device{}()}
    , distribution_{low, high}
    {
    }
    int operator()()
    {
        return distribution_(random_engine_);
    }
private:
    std::mt19937 random_engine_;
    std::uniform_int_distribution<int> distribution_;
};

//C++14
auto randomNumberBetween = [](int low, int high)
{
    auto randomFunc = [distribution_ = std::uniform_int_distribution<int>(low, high), 
                       random_engine_ = std::mt19937{ std::random_device{}() }]() mutable
    {
        return distribution_(random_engine_);
    };
    return randomFunc;
};

int main()
{
    std::vector<int> numbers;
    std::generate_n(std::back_inserter(numbers), 500, RandomNumberBetween(1, 100));
    // or ranges::generate(numbers, RandomNumberBetween(1, 100));

    for (int number : numbers)
    {
        std::cout << number << ' ';
    }
}
```

## You may also like

- [How to split a string in C++](https://www.fluentcpp.com/2017/04/21/how-to-split-a-string-in-c/)
- [How to reorder a collection with the STL](https://www.fluentcpp.com/2018/04/20/ways-reordering-collection-stl/)