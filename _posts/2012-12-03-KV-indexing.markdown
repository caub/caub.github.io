---
layout: post
title: Indexing data with key-value stores
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

Thus, we don't exceed *Sum(i=0..min(3,k) of Binomial Coefficient(i, k)) insertions*, a more reasonnable number. Actually there are high chances that rows like **theme=nature** or **title=the** already exist.

How we search it
-----------
{% highlight javascript %}
{
  'theme': ['nature'], 'title': ['brown', 'quick']
}
{% endhighlight %}
will be a query matching our example above, thanks to the row: 'theme=nature&title=brown&title=quick'.

As mentionned, for larger composites exceeding 3 elements, the first 3 elements will be queried directly against the database, the next 3 also, and so on.. then the intersection is done between the result sets document ids.

You can also form 'OR' queries:

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

 * Limitated (no search patterns)  
   * However we could expand searched rows with patterns like 'blu%' to cover ['blua', 'blub', 'bluc',...], (methods known as reverse regex). Or, more like Lucene, we could break large words into smaller one that we index also.
 * Fast: O(1) when the index exists (under 3 'AND')
 * Easy to use and manage, metadata are stored with the content to remove all indexes, if necessary
 * Other notes: the title sentence of the example was indexed with single words. To index larger specific expressions such as "brown fox", it's often preferable to post-filter out results like "brown cat and gray fox" rather than trying to index ordered combinations of title terms...

Appendix on numbers
-------

The problem with numbers is we want to search over a range, what can't give a raw hashtable.

A solution could be to work with a sorted-hashtable. But, else without it, you could try to index as key only the integer part, then you would fetch a set of keys. This solution becomes quickly messy.

A probably clean but complex way to deal with numbers is to use a hexadecimal-partitionning:
Let's take (int16) integers, we equally map this set to hexadecimals, (thus 8&rarr;`[0,2^13[`, 9&rarr;`[2^13,2^14[,` and so on), we iterate for the next resolution, ( 80&rarr;`[0,2^9[`, ... 8f&rarr;`[15*2^9,2^13[`, ...).
The partitioning is infinite, but we don't need to access to very low resolutions. So for storing a number, you will store its first k mappings, and to retrieve numbers between m and n, you will search the mappings near between them with range resolution close to (n-m)/2 or less.

Example of a range `[500, 750]`:  
the gap of 250 needs a resolution of less than 512=2^9  
thus the 8 sets `[80f,810,811,...,817]` (`[480,768]`) fit well as a search

The same way, [GeoCells](http://code.google.com/p/geomodel/source/browse/trunk/geo/geocell.py) enables to store 2D geographic positions, by splitting the globe surface in 16 areas and so on..

As a short example:
{% highlight javascript %}
[
  {'label':['hello','world'], 'position':['c']},
  {'label':['hello','world'], 'position':['ca']},
  {'label':['hello','world'], 'position':['cab']},
  {'label':['hello','world'], 'position':['cab2']},
  {'label':['hello','world'], 'position':['cab2d']},
  {'label':['hello','world'], 'position':['cab2d7']},
  // .. matches position (43.3, 7) as deep as you want resolution accuracy
]
{% endhighlight %}

For searching in the certain area, say [(43, 7), (44, 8.8)], it comes down to find a best resolution and the cells that cover this area:

{% highlight javascript %}
[
  {'position':['cab2']}, {'position':['cab3']}, {'position':['cab6']},
  {'position':['cab8']}, {'position':['cab9']}, {'position':['cabc']},
  // covers the area between latitudes 43 and 44, and longitudes 7 and 8.8
]
{% endhighlight %}
