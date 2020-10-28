---
title: "Match as an alternative to merge in R"
description: "While 'merge' function is the most straightforward solution to joining datasets by a common variable in R, sometimes 'match' is more intuitive."
tags: [R]
category: data
---



One of the most common operations in data wrangling is joining two sets of data by a common variable. Probably the most popular method for this is the obscure `vlookup` function in Excel (simply because it's the most widely used software for data manipulaton). The closest alternative in base R is `merge` and the dplyr package contains the join function family which is even more convenient. But there is a more simple and direct solution when only one variable needs to be added to a dataset. 

Suppose we have two data frames, `df.a` and `df.b`, and we wish to get the values of `other.var` from `df.b` into `df.a` so that each `id` gets their "own" value. There are [various methods for joining](https://www.google.ee/search?q=data+join), each yielding a different result. But in my experience *left join* on a single variable is the most frequent and this is what we will explore here.


{% highlight r %}
df.a <- data.frame(id = sample(LETTERS, 10), 
                   some.var = rnorm(10))
df.a
{% endhighlight %}



{% highlight text %}
##    id    some.var
## 1   K  0.18680978
## 2   W -1.20721078
## 3   P -0.05905494
## 4   L  1.55776950
## 5   Y  2.13328219
## 6   B  0.69926471
## 7   A -0.03656637
## 8   S -0.69594103
## 9   T  0.90289095
## 10  X  0.31828502
{% endhighlight %}



{% highlight r %}
df.b <- data.frame(id = sample(LETTERS, 20), 
                   other.var = runif(20, 1, 100))
df.b
{% endhighlight %}



{% highlight text %}
##    id other.var
## 1   T 18.461094
## 2   M 86.746373
## 3   N 76.536230
## 4   P 79.338731
## 5   B 62.402434
## 6   I 50.785243
## 7   O 65.291921
## 8   X 27.777302
## 9   A 65.784727
## 10  D 45.353893
## 11  H 26.760644
## 12  Q 10.775249
## 13  E 47.873233
## 14  C 74.418394
## 15  Z 67.870696
## 16  Y 35.722271
## 17  K 40.220227
## 18  U 69.011976
## 19  V  3.544297
## 20  F  5.644257
{% endhighlight %}

# Left join with 'merge'

When using `merge`, we specify the arguments of the function, run it and then through some *magic* a new dataset with requested columns is created. Note that we don't need to specify by which variable we wish to merge if variable names are the same.


{% highlight r %}
df.merge <- merge(df.a, df.b, all.x = T, all.y = F)
df.merge
{% endhighlight %}



{% highlight text %}
##    id    some.var other.var
## 1   A -0.03656637  65.78473
## 2   B  0.69926471  62.40243
## 3   K  0.18680978  40.22023
## 4   L  1.55776950        NA
## 5   P -0.05905494  79.33873
## 6   S -0.69594103        NA
## 7   T  0.90289095  18.46109
## 8   W -1.20721078        NA
## 9   X  0.31828502  27.77730
## 10  Y  2.13328219  35.72227
{% endhighlight %}

# Left join with 'match'

A more hands-on approach involves first figuring out which rows in `df.a` correspond to which rows in `df.b` according to `id`. The `match` function allows us to do just that.


{% highlight r %}
match(df.a$id, df.b$id)
{% endhighlight %}



{% highlight text %}
##  [1] 17 NA  4 NA 16  5  9 NA  1  8
{% endhighlight %}

Now that we have the row numbers, we can simply return `other.var` in `df.b` where the matches occur. A useful side effect is that we can define the name for the new variable while matching.


{% highlight r %}
df.a$other.var <- df.b$other.var[match(df.a$id, df.b$id)]
{% endhighlight %}

Now let's compare the results.


{% highlight r %}
df.merge
{% endhighlight %}



{% highlight text %}
##    id    some.var other.var
## 1   A -0.03656637  65.78473
## 2   B  0.69926471  62.40243
## 3   K  0.18680978  40.22023
## 4   L  1.55776950        NA
## 5   P -0.05905494  79.33873
## 6   S -0.69594103        NA
## 7   T  0.90289095  18.46109
## 8   W -1.20721078        NA
## 9   X  0.31828502  27.77730
## 10  Y  2.13328219  35.72227
{% endhighlight %}



{% highlight r %}
df.a
{% endhighlight %}



{% highlight text %}
##    id    some.var other.var
## 1   K  0.18680978  40.22023
## 2   W -1.20721078        NA
## 3   P -0.05905494  79.33873
## 4   L  1.55776950        NA
## 5   Y  2.13328219  35.72227
## 6   B  0.69926471  62.40243
## 7   A -0.03656637  65.78473
## 8   S -0.69594103        NA
## 9   T  0.90289095  18.46109
## 10  X  0.31828502  27.77730
{% endhighlight %}

We can see that the result is essentially the same. What `merge` has done is rearranged the rows which is something we might not want to happen. So I encourage the use of `match` when possible since it allows the addition of a single column without running a function over entire data sets.
