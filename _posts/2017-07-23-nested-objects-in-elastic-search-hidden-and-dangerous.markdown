---
layout: post
title:  "Nested objects in ElasticSearch: hidden and dangerous"
date:   2017-07-23 16:24:50 +0300
categories: Dev
---
Last week it took me a lot of time to debug the broken query that depends on array of objects. The problem was really simple but I end up in reindexing all data. It can be easily avoided in case if you know about one exception in the rule of working with arrays in ES.

To create an index in ES a mapping with declaration of fields and their datatypes should be provided. Regardless a datatype, any field can be treated as an array of elements with selected type. For example, if field `tag` is declared as `text` datatype, you can store there both a single string or an array of strings. There is no need in any additional actions to search a value in array. The query:

{% highlight json %}
{
  "query": {
    "match": {
      "tag": "film"
    }
  }
}
{% endhighlight %}

matches both of the following documents:

{% highlight json %}
{ "tag" : "film" }
{ "tag" : ["film", "grain"] }
{% endhighlight %}

However it doesn’t work that way with objects. In fact ES also can handle `object` datatype as an array of objects, but they can’t be queried independently as separate objects by default. And here comes my problem.

Let assume that we have nested objects with two fields: first of them contains tag's name that describes a document and the second one contains probability value for this tag (0-100). For example: `[{ "name" : "dog", "probability" : 93 }, { "name" : "fur", "probability" : 80 }]`. I need to match all documents that have a tag with probability greater or equal to 90. Based on the assumption that ES will store objects as an array I’ve written a query:


{% highlight json %}
{
  "bool": {
    "must": [
      { "match": { "tag.name" : "fur" } },
      { "range": { "tag.probability" : { "gte" : 90 } } }
    ]
  }
}
{% endhighlight %}


The `bool must` syntax means that both of nested conditions must be true. I expected that this query should match documents that have object with `tag == fur` and `probability >= 90`. However a document from the example was matched despite the probability value of 'fur' tag is 80.
{% highlight json %}
{
  "tag" : [
    { "name" : "dog", "probability" : 93 },
    { "name" : "fur", "probability" : 80 }
  ]
}
{% endhighlight %}

After several debugging sessions I found out that ES flattens array of objects to many arrays of separate fields by default. The previous documents is presented as:
{% highlight json %}
{
  "tag.name" : ["dog", "fur"],
  "tag.probability": [93, 80]
}
{% endhighlight %}
The query matches the document because it includes 'fur' in `tag.name` and there is a value greater than 90 in the `tag.probability` field. This behaviour isn't correct for my case.


To query objects entirely in ElasticSearch, array of objects should be indexed with [nested datatype](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html).
In this case each object is stored as a separate hidden document and can be queried independently with a [nested query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html):
{% highlight json %}
{
  "nested": {
    "path": "tag",
    "query": {
      "bool": {
        "must": [
          { "match": { "tag.name": "fur" }},
          { "match": { "tag.probability":  { "gte": 90 }}}
        ]
      }
    }
  }
}
{% endhighlight %}
This way gives a correct result, but if you have already set datatype to `object` or you have `dynamic mapping` enabled (it also maps datatype to `object`) then you can’t just update your mapping and you are doomed to reindex all data from scratch.

At this point I thought that my problem is solved. I was naive. After several days of reindexing with `nested object` datatype I've realized that number of documents in the index has been increased by 700%, also a disk space consumed by the index has been increased by 130%. These new mysterious documents are hidden nested objects. Even without them I have a large amount of data stored in the index. But the most terrible consequence was a performance regression. Search queries took almost 10 times longer than before reindexing.

Such performance regression was unacceptable for me, so I came up with another solution. Since a threshold for `tag.probability` was fixed for all queries and I use another database as a main storage, it allowed me to move the part of query with `tags.probability >= 90` to an indexing stage. I decided to declare a field `trusted_tags` in a mapping and serialize there only tags that have `probability >= 90`. With the help of this approach there is no need for additional `probability` condition in a query, correct documents will be matched with:

{% highlight json %}
{
  "query": {
    "match": {
      "trusted_tags": "fur"
    }
  }
}
{% endhighlight %}

Despite it works like a charm, I had to reindex all data from scratch again. In the end, I categorically wouldn't recommend to use `nested object` datatype if number of nested documents is bigger than number of actual indexed documents.
