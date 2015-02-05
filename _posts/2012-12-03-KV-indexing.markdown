---
layout: post
title: Indexing data on key-value stores
excerpt: Some thoughts on indexing data with simple hashtables.
---

Indexing is a way to retrieve data by some keywords and metadata. Most databases nowadays include indexation with various techniques: Balanced trees, Lucene indexes, ...

For this example let's consider a storage at its simplest or almost, a hashtable.

The specifications are to be able to search a data by its metadata (name-value) pairs. To be concrete, here's an example of metadata:
 
{% highlight javascript %}
{
  'id': 'example_id',
  'metadata': {
    'auhor': ['mike123@gmail.com'],
    'title': ['the','quick','brown','fox','jumps','over','lazy','dog'],
    'date': ['2012-11-12'],
    'theme': ['nature', 'animals']
  }
}
{% endhighlight %}

Thus, we would like our search to be able to find **ids** from the keywords *author*: *someone*, and *title* containing *something*.

For this, we will rely on [inverted index](http://en.wikipedia.org/wiki/Inverted_index) technique and generate many key-value pairs such as:

{% highlight javascript %}
{
  'auhor=mike123%40gmail.com'                           : ['example_id'] 
  'title=quick'                                         : ['example_id']
  'title=brown'                                         : ['example_id']
  'date=2012-11-12'                                     : ['example_id']
  'theme=nature'                                        : ['example_id']
  'theme=animals'                                       : ['example_id']
  //...
{% endhighlight %}

and also composition between name-value pairs with '&':

{% highlight javascript %}
  //...
  'auhor=mike123%40gmail.com&title=quick'               : ['example_id']
  'auhor=mike123%40gmail.com&theme=nature'              : ['example_id']
  'title=dog&title=quick'                               : ['example_id']
  //..
  'auhor=mike123%40gmail.com&theme=animals&title=quick' : ['example_id']
  //...
  //...
}
{% endhighlight %}

... but we would end up with 2^k insertions (k= sum of metadata values size, 12 here), so we limit composite length with min(**3**, k). (**3** is arbitrary here, this parameter should just be a small number)

Thus, we don't exceed `Sum(i=0..min(3,k) of Binomial Coefficient(i, k)) insertions`, a more reasonnable number. Actually there are high chances that rows like **theme=nature** or **title=the** already exist in database and get reused for other documents.

How we search it
-----------
{% highlight javascript %}
{
  'theme': ['nature'], 'title': ['brown', 'quick']
}
{% endhighlight %}
will be a query matching our example above, thanks to the row: 'theme=nature&title=brown&title=quick'.

As mentionned, for larger composites exceeding 3 elements, the first 3 elements will be queried directly against the database, the next 3 also, and so on.. then the intersection is done between the result sets containing document ids.

You can also form 'OR' queries, that perform parallel queries and merge results in the same map:

{% highlight javascript %}
[
  {'theme':['nature'], 'title':['brown','quick']},
  {'theme':['nature'], 'title':['test']},
  {'theme':['birds']}
]
{% endhighlight %}

Will match again the above example document, or also ones themed with nature and title containing 'test', or themed with birds.

In summary
---------

This method of indexation has the following characterictics:

 * Restricted (no search patterns)  
   * However we could expand searched rows with range (see appendix on numbers) or patterns like 'blu%' to cover ['blua', 'blub', 'bluc',...], (methods known as reverse regex). We could break large words into smaller one that we index also, semantic parsing like stemming, de-conjugate verbs, eliminate plurals... similarly to Lucene text-processing.
 * Fast: O(1) when the index exists (under 3 'AND')
 * Easy to use and manage, metadata are stored with the content to remove,update or reinsert all indexes, if necessary
 * Other notes:
   * The title sentence of the example was indexed with single words. To index larger specific expressions such as "brown fox", it's often preferable to post-filter out results rather than trying to index ordered combinations of title terms...
   * We can optionally define a handler name in the search query, and in the insert query, that generates indexes differently, for example *text* (by default, does indexation like we saw, by splitting in words), *binary* (no indexation on the specified binary field), *range1/range2/range3/range4...* (range handlers, see below).

Appendix on numbers
-------

The problem with numbers is we often want to search them over a range.

A solution could be to work with a sorted-hashtable, it would homever serve only for one dimension at a time. One could also try to index as key only the integer part, then you would fetch a set of keys for the first decimals, etc... This solution is a bit messy but is closer to the goal.

One idea with numbers ranges is to use a partitionning, let's consider the set of Int16 numbers [-32768, 32767]. They will be partitionned given a particular 'resolution', for example resolution 1 deals with sub-ranges of size 10000, -4 maps [-32768,-30000[, -3 maps [-30000, -20000[... Resolution 2 have ranges of 1000, -33 maps [-32768,-32000[, -32 maps [-32000, -31000[
Indexing 1337 with resolution 3 gives 3 indexes, 0, 01 and 013.
Searching the range [1960, 2015] with resolution 3 gives the indexes 019 and 020 that covers [1900,2100[, with reolution 4 gives the indexes [0196,0197,0198,0199,0200,0210] covering [1960, 2020]. A post-filtering refines the borders of the search interval.

For 2 or more range variables (2D for example), we can continue to apply this method for each one.
