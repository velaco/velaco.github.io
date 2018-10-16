---
layout: post
title: Probability of Getting Zero Points in Dr. Burnörium's Psycho Dice Game 
authors: Aleksandar Ratesic
tags: [R, igraph, path-analysis]
excerpt_separator: <!--more-->
---

![Psycho Dice Game]({{ site.baseurl }}/images/psycho-dice/PsychoDiceCover.jpg){: style="display:block;margin-left:auto;margin-right:auto"}

"What is the probability of getting a score of zero in Dr. Burnörium's [Psycho Dice Russian Roulette Game](https://amzn.to/2KQNq6B){: target="_blank"}?" That question came to mind after a Psycho Dice gaming session, during which we noticed that turns ending with zero points occur about a quarter of the time.

<!--more-->

## About the Game

### Goal

Your goal is to get the highest score after six rounds of rolling dice - or at least to avoid getting the lowest score because the loser has to eat a Chocolate Bullet. Although some are harmless and sweet, the majority contain capsaicin and are painful to eat. 

### Scoring Rules

Points in the game are won by throwing dice. You start your roll in each turn with five dice, which contain four numbers and two bullets. If you roll any bullet dice, the roll will be non-scoring. Bullet dice are discarded in the next roll, so if two of your five dice show a bullet for example, you will throw three dice in your next roll. Your turn ends when all five dice are discarded.

Because any roll containing a bullet dice is non-scoring, there are several different ways to get to zero dice without scoring any points.

## Possible Paths to Zero Points

I used R to generate the probabilities of rolling between 0 and N bullets, where N is the number of dice in a roll. However,  

```r
> head(dfEdge)
  Source Target Probability
1      1      1       0.667
2      1      0       0.333
3      2      2       0.444
4      2      1       0.444
5      2      0       0.111
6      3      3       0.296
```

Then I used the [igraph](http://igraph.org) package to create a graph of possible paths from rolling 5 dice to 0 dice `graph_from_data_frame()`

The nodes in the graph below represent the number of dice in the current roll, wheres the edges are paths to the number of dice in the next roll. Each edge is associated with the probability of taking that path. For example, if you throw five dice and get one bullet for example, you'll move from node 5 to node 4. (There is a 32.9% chance of that happening.) 

![Psycho Dice 0-score Probability Graph]({{ site.baseurl }}/images/psycho-dice/PsychoDice.png){: style="display:block;margin-left:auto;margin-right:auto"}

Each node also has a loop because you'll throw the same number of dice again if you don't get a bullet dice. The probability of scoring points is the lowest when throwing five dice, 13.2%, but the probability of scoring points increases to 66.7% by the time you're left with one die.

Running the `igraph::all_simple_paths()` function returned 16 paths to zero dice without scoring a single point.

|**Path to 0 points**     |**Probability**|
|:--------------------------|-----------:|
|5 -> 1 -> 0                |   0.0136530|
|5 -> 2 -> 1 -> 0           |   0.0243956|
|5 -> 2 -> 0                |   0.0183150|
|5 -> 3 -> 1 -> 0           |   0.0243217|
|5 -> 3 -> 2 -> 1 -> 0      |   0.0215976|
|5 -> 3 -> 2 -> 0           |   0.0162144|
|5 -> 3 -> 0                |   0.0121730|
|5 -> 4 -> 1 -> 0           |   0.0108461|
|5 -> 4 -> 2 -> 1 -> 0      |   0.0143984|
|5 -> 4 -> 2 -> 0           |   0.0108096|
|5 -> 4 -> 3 -> 1 -> 0      |   0.0096071|
|5 -> 4 -> 3 -> 2 -> 1 -> 0 |   0.0085311|
|5 -> 4 -> 3 -> 2 -> 0      |   0.0064047|
|5 -> 4 -> 3 -> 0           |   0.0048083|
|5 -> 4 -> 0                |   0.0039480|
|5 -> 0                     |   0.0040000|
|----
|*TOTAL*                    | *0.2040236*|
{: class="table-striped" style="margin-left:auto;margin-right:auto"}

All throws are independent because the number of bullets you get 

The only remaining thing to do is to 
