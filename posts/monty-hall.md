---
title: The Monty Hall Problem
tags:
  - blog
emoji: ""
link: ""
toc: true
date: "2024-05-03"
---

## Background

The Monty Hall problem is a puzzle that's rooted in probability and is a bit of a brain
teaser. A friend recently described the problem and that got me thinking about how
to simulate the puzzle and some of the common gotchas that trip people up.

### The problem

The problem described is one of a game. You have 3 doors to choose from,
2 of which have goats behind them and 1 has a car behind it. You win whatever is
behind the door that you choose. Once you choose a door, the host will tell you
about one door that has a goat behind it. Once the host tells you this information,
you have the choice to stay at the door you've already chosen or change doors. Which
is better?

### Assumptions

According to Wikipedia, there are a few assumptions:

> [Monty Hall problem > Standard Assumptions](https://en.wikipedia.org/wiki/Monty_Hall_problem#Standard_assumptions)
> 1. The host must always open a door that was not selected by the contestant.
> 2. The host must always open a door to reveal a goat and never the car.
> 3. The host must always offer the chance to switch between the door chosen originally and the
> closed door remaining.

## Simulation

Therefore, we can come up with a simple simulation in Python:

```python
import random


def main():
    """
    Run the main simulation 100000 times and print the number of times
    switching the door was better and the number of times switching the door was worse
    """

    # Run the simulation 100000 times and save the results of each iteration
    res = []
    for _ in range(0, 100000):
        res.append(better())

    # Count the number of times switching doors was better
    good = res.count(True)

    # Count the number of times switching doors was worse
    bad = res.count(False)

    print(f"Number of times switching was good: {good} ({(good/len(res))*100})")
    print(f"Number of times switching was bad: {bad} ({(bad/len(res))*100})")


def better():
    """
    Test the Monty Hall problem to decide which option is better (switching doors or not)
    """

    # Choose where the car is
    car_pos = random.randint(0, 2)

    # Set a list where we know the goats are now
    goat_pos = [x for x in range(0, 3) if x != car_pos]

    # Randomly guess where the car might be
    guess_pos = random.randint(0, 2)

    # Choose one of the goat options to tell the player about
    # *IMPORTANT* It CANNOT be the door a player picked
    while True:
        told_goat_pos = random.choice(goat_pos)
        if told_goat_pos != guess_pos:
            break

    # Figure out the other option that we switch to
    other_opt = [
        x for x in range(0, 3) if x != guess_pos and x != told_goat_pos
    ][0]

    # Check if the other option is the car position. If it is, return True
    # meaning that the switch was better
    if other_opt == car_pos:
        return True
    return False


if __name__ == "__main__":
    main()
```

The result of running this simulation a few different times is as follows:

```text
$ python3 /tmp/test.py
Number of times switching was good: 66642 (66.642)
Number of times switching was bad: 33358 (33.358)

$ python3 /tmp/test.py
Number of times switching was good: 66721 (66.721)
Number of times switching was bad: 33279 (33.278999999999996)

$ python3 /tmp/test.py
Number of times switching was good: 66713 (66.713)
Number of times switching was bad: 33287 (33.287)
```

Which is pretty much what we expect. You have a 2/3 (~66%) chance of winning the car if you
change doors. If you don't change doors, you have a 1/3 (~33%) chance of winning the car.

A common misconception of people is this would just result in a 1/2 (50%) chance of getting the car. This is because once a goat is revealed, you have two options left
(one of which a goat, the other the car) which results in a 1/2 (50%) chance.

The one important thing that's missing from making the above true is that the host never
reveals to the player what is behind the player's chosen door (based on our assumptions).
Therefore, you can't operate on the assumption that our chances are 50/50. Here's the same simulation code, but we randomly tell the player about which door a goat is behind:

```python
import random


def main():
    """
    Run the main simulation 100000 times and print the number of times
    switching the door was better and the number of times switching the door was worse
    """

    # Run the simulation 100000 times and save the results of each iteration
    res = []
    for _ in range(0, 100000):
        res.append(better())

    # Count the number of times switching doors was better
    good = res.count(True)

    # Count the number of times switching doors was worse
    bad = res.count(False)

    print(f"Number of times switching was good: {good} ({(good/len(res))*100})")
    print(f"Number of times switching was bad: {bad} ({(bad/len(res))*100})")


def better():
    """
    Test the Monty Hall problem to decide which option is better (switching doors or not)
    """

    # Choose where the car is
    car_pos = random.randint(0, 2)

    # Set a list where we know the goats are now
    goat_pos = [x for x in range(0, 3) if x != car_pos]

    # Randomly guess where the car might be
    guess_pos = random.randint(0, 2)

    # Randomly tell the player about what door has a goat behind it
    told_goat_pos = random.choice(goat_pos)

    # Figure out the other option that we switch to
    other_opt = [
        x for x in range(0, 3) if x != guess_pos and x != told_goat_pos
    ][0]

    # Check if the other option is the car position. If it is, return True
    # meaning that the switch was better
    if other_opt == car_pos:
        return True
    return False


if __name__ == "__main__":
    main()
```

The result of running this modified simulation a few different times is as follows:

```text
$ python3 /tmp/test.py
Number of times switching was good: 50115 (50.114999999999995)
Number of times switching was bad: 49885 (49.885000000000005)

$ python3 /tmp/test.py
Number of times switching was good: 50234 (50.234)
Number of times switching was bad: 49766 (49.766)

$ python3 /tmp/test.py
Number of times switching was good: 49879 (49.879)
Number of times switching was bad: 50121 (50.121)
```

Which results in the 50/50 probability we expected! Pretty neat!