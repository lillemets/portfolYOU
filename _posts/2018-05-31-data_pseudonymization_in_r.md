---
title: "Data pseudonymization in R"
description: With the recent adoption of GDPR, pseudonymization is now a relevant method of protecting personal data. An example of how to pseudonymize data in R is given.
tags: [R, Methods]
category: data
---



The General Data Protection Regulation (GDPR) recently enforced in EU requires that personal data should be either anonymized or pseudonymized when used for other purposes than initially intended. I always knew pseudonymization was a thing but I was unaware of the fancy term until attending [OpenAIRE seminar on research data](https://utlib.ut.ee/en/how-get-maximum-research-data). Unlike the term, it's actually a simple procedure and can be implemented in R. But how exactly?

To begin with, we need a suitable dataset. We'll use the first 5 rows of the `mtcars` dataset and add an identifier column *name*. The first row is duplicated in order to demonstrate the process in case of multiple observations of the same subject.


{% highlight r %}
cars <- data.frame(name = rownames(mtcars), mtcars, row.names = NULL)[1:5, ]
cars <- rbind(cars, cars[1, ])
cars
{% endhighlight %}



{% highlight text %}
##                name  mpg cyl disp  hp drat    wt  qsec vs am gear carb
## 1         Mazda RX4 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
## 2     Mazda RX4 Wag 21.0   6  160 110 3.90 2.875 17.02  0  1    4    4
## 3        Datsun 710 22.8   4  108  93 3.85 2.320 18.61  1  1    4    1
## 4    Hornet 4 Drive 21.4   6  258 110 3.08 3.215 19.44  1  0    3    1
## 5 Hornet Sportabout 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2
## 6         Mazda RX4 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
{% endhighlight %}

Data anonymization is seemingly simple: we just need to remove the column called *name*. This is not always the case though, since it might still be possible to identify subjects by values. As a general recommendation, it should be impossible to narrow down each subject in pseudonymized data to less than 5 subjects. So this is actually more complicated and will not be discussed any further here.


{% highlight r %}
cars[, grep('name', names(cars), inv = T)]
{% endhighlight %}



{% highlight text %}
##    mpg cyl disp  hp drat    wt  qsec vs am gear carb
## 1 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
## 2 21.0   6  160 110 3.90 2.875 17.02  0  1    4    4
## 3 22.8   4  108  93 3.85 2.320 18.61  1  1    4    1
## 4 21.4   6  258 110 3.08 3.215 19.44  1  0    3    1
## 5 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2
## 6 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
{% endhighlight %}

Pseudonomization is less extreme. The idea is to replace identifiers (here the *name* column) with aliases (pseudonyms/tokens) so that it is possible to recreate the original dataset when necessary. In R we can generate a good alias for instance by taking a random subset of 8 from all the (uppercase) letters and numbers and then collapsing the result into a single string. 


{% highlight r %}
keyLen <- 8 # Define the length of key
paste(sample(c(LETTERS, 0:9), keyLen, replace = T), collapse = '')
{% endhighlight %}



{% highlight text %}
## [1] "BEN4GIVU"
{% endhighlight %}

However, there are several considerations when generating aliases for actual data:

1. Aliases should be random and not sequential because the ordering of rows in data usually follows a pattern. In case of survey data, this may be the time of submitting a response which may enable the re-identification of subjects. Non-sequential aliases ensure that once rows are shuffled, the original ordering cannot be restored (unless the original ordering is present in some other variable).
2. Each row should usually have a unique identifier. However, this might not be true for longitudinal data in long format where each subject is represented by multiple rows.
3. Different subjects should never be assigned identical aliases. When generating aliases as in the example above, there are 2 821 109 907 456 unique permutations possible. So while duplicate aliases are extremely unlikely, an alias generation algorithm must make them inherently impossible. 

Taking these considerations into account, we can use the function below to generate a lookup table that associates each unique identifier to an alias.


{% highlight r %}
makeLookupTable <- function(data, id.var, key.length) {
  if (anyDuplicated(data[, id.var])) warning('Duplicate id values in data.')
  aliases <- c(1,1) # Allow the while loop to begin
  while (any(duplicated(aliases))) { # Loop until all keys are unique
    aliases <- replicate(length(unique(data[, id.var])), 
                         paste(sample(c(LETTERS, 0:9), key.length, replace = T), collapse = ''))
  }
  lookup.table <- data.frame(id = unique(data[, id.var]), key = aliases)
  return(lookup.table)
}
{% endhighlight %}

To tackle the first issue, the function will prompt a warning, leaving the user to decide whether duplicate identity values should be dealt with or not. Then, for each unique identifier an alias is generated and this process is iterated until all keys are unique. More than one iteration is rarely necessary when keys are not too short. Finally, the result is returned as a `data.frame` object.

Once generated, the lookup table can be used to replace names with aliases or vice versa using the following functions.


{% highlight r %}
# Replace names with aliases
addAlias <- function(data, id.var, lookup.table) {
  data[, id.var] <- lookup.table[, 'key'][match(data[, id.var], lookup.table[, 'id'])]
  return(data)
}

# Replace aliases with names
replaceAlias <- function(data, id.var, lookup.table) {
    data[, id.var] <- lookup.table[, 'id'][match(data[, id.var], lookup.table[, 'key'])]
    return(data)
}
{% endhighlight %}

Below is an illustration how these functions work in practice. Note that the duplicate row prompts a warning and only unique identifiers are written to the lookup table.


{% highlight r %}
# Generate a lookup table
(aliasLookup <- makeLookupTable(cars, 'name', 8))
{% endhighlight %}



{% highlight text %}
## Warning in makeLookupTable(cars, "name", 8): Duplicate id values in data.
{% endhighlight %}



{% highlight text %}
##                  id      key
## 1         Mazda RX4 M0HT4G7Y
## 2     Mazda RX4 Wag CBLQVX40
## 3        Datsun 710 P1VWUCG8
## 4    Hornet 4 Drive LIB635UA
## 5 Hornet Sportabout IDWULIXC
{% endhighlight %}



{% highlight r %}
# Replace names with aliases
(carsAliases <- addAlias(cars, 'name', aliasLookup))
{% endhighlight %}



{% highlight text %}
##       name  mpg cyl disp  hp drat    wt  qsec vs am gear carb
## 1 M0HT4G7Y 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
## 2 CBLQVX40 21.0   6  160 110 3.90 2.875 17.02  0  1    4    4
## 3 P1VWUCG8 22.8   4  108  93 3.85 2.320 18.61  1  1    4    1
## 4 LIB635UA 21.4   6  258 110 3.08 3.215 19.44  1  0    3    1
## 5 IDWULIXC 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2
## 6 M0HT4G7Y 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
{% endhighlight %}



{% highlight r %}
# Replace aliases with names which returns the original data
(carsNames <- replaceAlias(carsAliases, 'name', aliasLookup))
{% endhighlight %}



{% highlight text %}
##                name  mpg cyl disp  hp drat    wt  qsec vs am gear carb
## 1         Mazda RX4 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
## 2     Mazda RX4 Wag 21.0   6  160 110 3.90 2.875 17.02  0  1    4    4
## 3        Datsun 710 22.8   4  108  93 3.85 2.320 18.61  1  1    4    1
## 4    Hornet 4 Drive 21.4   6  258 110 3.08 3.215 19.44  1  0    3    1
## 5 Hornet Sportabout 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2
## 6         Mazda RX4 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
{% endhighlight %}

When it is assured that names can not be deduced from the values, the `carsAliases` table is now pseudonymized and can be published (provided that `aliasLookup` is stored with restricted access).
