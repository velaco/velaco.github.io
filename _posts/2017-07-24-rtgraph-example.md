---
layout: post
title: RtGraph Example - Graphing How a Tweet Can Spread Among Retweeters
authors: Aleksandar Ratesic
tags: [R, igraph, Twitter, twitteR]
excerpt_separator: <!--more-->
---

“How far can users' tweets spread thanks to their followers' retweets?” I asked myself one day. It's important to note that I was thinking in terms of degrees of separation rather than geographic distances. For example, if one of your followers retweets something you wrote, their followers will see that tweet and are now two steps away from you. If one of them does not follow you, but decides to retweet it further, their followers will be three steps away from you. And so on...

To satisfy my curiosity, I wrote an R script that would allow me to graph the relationships between an author of a tweet and its retweeters. The script I used in this example is available at the [RtGraph repository](https://github.com/velaco/rtgraph){:target="blank"}. <!--more-->

## Selecting a Tweet

In some alternate universe, a popular version of me has a large and active Twitter following. However, that’s not the case in this universe, so I had to use an account belonging to another person or organization to carry out this example. I wrote the script using [RStudio](https://www.rstudio.com/){:target="blank"}, so I thought it would be appropriate to use one of the tweets from their account. 

Twitter’s REST API allows you to collect 100 of the most recent retweets and retweeters, so I looked through their timeline to find a tweet with more than 100 retweets for the analysis. Here it is:

![Tweet Selection]({{ site.url }}/images/rt-graph/tweetContent.png)

The script needs the number (ID) of the tweet, which is found in the URL of the tweet:

![Tweet ID]({{ site.url }}/images/rt-graph/tweetURL.png)

## Data Collection and Output

Once the script has a valid tweet ID to work with, it collects the IDs of the author's followers, the IDs of retweeters, and the IDs of each retweeter's followers.

Although this tweet had 191 retweets at the time, only 98 user IDs were received. Seven users were not found (404 error) when a list of their followers was requested, so the final number of retweeters included in the graph was 91. Seventy-one of those users followed RStudio on Twitter.

The script stores the retweeters' and their followers' IDs as a dataset of edges. In this example, the original dataset contained 32,913 edges. However, my aim was to plot only the relationships between retweeters, so for the final edge graph, the script selects only those edges that connect two retweeters. That brought the number of edges between retweeters down to 86, so the number of edges in the final graph was 157 when you include the edges between RStudio and their followers who retweeted the tweet.

The final output includes two datasets, one for the edges and one for the nodes of the graph, as well as an image that shows the relationships between the author and the retweeters. The script also saves all objects into an RData file so there is no need to repeat API requests to retrieve data that has already been collected.

### Datasets

The edge dataset consists of the *Source* user ID and the *Target* user ID. The *Target* user is the follower of a *Source* user, so the arrows of the edges show possible paths for a tweet to reach users, even if they are not following the author of the tweet.

```r
> head(retweetersEdge)
     Source             Target
1 235261861 818598351570608128
2 235261861           11953392
3 235261861         2908869704
4 235261861         3288057307
5 235261861         2474106733
6 235261861         3363881602
```

Note that the names *Source* and *Target* do not mean the igraph object must be a directed graph. You can make the edges undirected by setting the **graph_from_data_frame()** function's **directed** argument to FALSE.

The node dataset is required to associate each vertex in the graph with a user ID and optional attributes. In this case, the only attribute I used is *Following* to determine if the retweeters also follow the author of a tweet. The label "Author" is reserved for the author.

```r
> head(retweetersNode, n = 10)
                   ID Following
1           235261861    Author
2          3288057307       Yes
3          3363881602       Yes
4           881530736       Yes
5          2289169778       Yes
6  771530276488908800       Yes
7          2875208193        No
8            45518340       Yes
9           155918866        No
10          282980773        No
```

I used the *Following* attribute to color each node because that makes it easier to determine which retweeters follow the author and which ones don't.

```r
V(rtsNetwork)$color <- ifelse(V(rtsNetwork)$Following == "Author",
                              "black",
                              ifelse(V(rtsNetwork)$Following == "Yes",
                                     "green",
                                     "red")
                              )
```

## Visualization

The black node in the middle of the graph is the tweet and the arrows show in which directions it can spread among retweeters. Green nodes represent the retweeters who are also the author's followers, whereas the red nodes are retweeters who do not follow the author on Twitter.

![Retweeters]({{ site.url }}/images/rt-graph/884795477178421248-RtGraph.png)

In this example, 20 users retweeted the tweet even though they do not follow RStudio on Twitter. Fourteen of those users were able to see and retweet RStudio's tweet because they followed another retweeter.

Six retweeters were not linked to the author at all. My first guess was that they unfollowed RStudio or some of the other retweeters, but then I remembered that the API request came back with only 98 user IDs from a total of 191 retweeters. It's possible that the tweet reached those users through one or more retweeters whose IDs were not collected. I also considered the possibility that those users are following someone who only liked the tweet, but did not retweet it. Although I was tempted to expand the script and collect the IDs of users who liked the tweet, I decided to drop that possibility for now.

Two retweeters were three hops away from the author, which means that this tweet traveled at least four edge hops as it reached their followers. (An edge hop refers to the number of edges one would have to cross to travel from one vertex to another.) However, without a full list of users who retweeted and/or liked a tweet, it is impossible to determine the maximum number of hops a tweet traveled.

I found the edges of the graph interesting as well because some neighborhoods have a higher density than others. I was curious to see what a community detection algorithm would find, but I decided to leave that analysis for another time.

## Concluding Remarks

Unfortunately, this script can never produce a complete graph because of the rate limitations determined by Twitter's API. Twitter has a rate limit of 15 requests for follower IDs, after which the script needs to take a 15-minute break, so it takes a lot of time to collect the retweeters' user IDs. Furthermore, the API also returns a maximum of 100 retweeters' IDs, and even if there are more than 100 retweets, the request does not return the IDs from all retweeters. 

Despite those limitations, I was satisfied with the output because it was interesting to visualize how far a tweet can travel from the author through retweeters.

To view or download the script I used, visit the [RtGraph repository](https://github.com/velaco/rtgraph){:target="blank"}.

## Resources

1. Csardi, G., & Nepusz, T. (2006). The igraph software package for complex network research. *InterJournal Complex Systems, 1695*, 1-9.
2. Gentry, J. (2015). *twitteR: R Based Twitter Client.* [R package version 1.1.9.] Retrieved from https://CRAN.R-project.org/package=twitteR
3. Ognyanova, K. (2016). *Network analysis and visualization with R and igraph.* Retrieved from http://kateto.net/networks-r-igraph
