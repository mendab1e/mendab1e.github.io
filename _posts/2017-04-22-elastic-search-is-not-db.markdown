---
layout: post
title:  "Why one shouldn't use ElasticSearch as a data storage"
date:   2017-04-22 12:53:53 +0500
categories: Dev
---
Easy to notice that popularity of ElasticSearch has been growing fast. Almost every new project connected with full-text search and scoring prefers it to Sphinx. A lot of people, who are new to full-text searching frameworks, have tried ES in their projects. Last year I had a pleasure to drink a beer with two different groups of Ruby developers, in both of them someone said: “We have used ElasticSearch to store our data and we got so many troubles with it”. To be honest, generally it isn't a good idea to use ES as a main storage for your project. Let me explain why.

ElasticSearch is a search engine, it stores documents in indices across shards of a cluster's nodes. Indices represent collections of documents with similar characteristics. In ElasticSearch, document is a JSON object that contains different fields. ES can’t scan all documents in a sequence to handle search queries, because it would be too slow, so it uses a search index. To understand how it works you should know the basics of indexing.

The lower level of the search abstraction in ES is an inverted index. Inverted index is a data structure that stores a mapping between each words of all documents and documents that contain them. Let's assume we have these 3 sentences:

| id | sentence          |
|----|-------------------|
| 1  | This is my cat    |
| 2  | I like your cat   |
| 3  | Here is my number |

Now we need to lowercase words and build an inverted index for them:

| term   | sentence_id |
|--------|-------------|
| this   | 1           |
| is     | 1, 3        |
| my     | 1, 3        |
| cat    | 1, 2        |
| i      | 2           |
| like   | 2           |
| your   | 2           |
| here   | 3           |
| number | 3           |

For example, with the help of this index the word "cat" can be found in sentences 1 and 2. From this example it is clear that original data isn't used to find documents’ ids with inverted index. By default, ElasticSearch stores JSON data in the `_source` field and returns it on search request. Source data can be used for highlighting, reindexing or displaying to user. The `_source` parameter can be set to `false`, in this case data should be retrieved from different source. It is possible only if documents' ids in ES are matched with documents' ids in different storage. For example, one has queried "cat" and got ids: [1, 2] in the result. A separate query with ids 1 and 2 should be sent to a storage for documents. This way allows avoiding data duplication and save some space, but has a little overhead on an additional query.

Technically ElasticSearch can be used as a main data storage, but it has some unpleasant disadvantages.

To make your data searchable, ElasticSearch needs to know what type of data each field contains and how it should be indexed. The process of defining a schema with data types and indexing algorithms is called mapping. A schema should be provided during creation of an index. When you pass your data to the ES-index, Lucene, that is located under the hood of ES, builds many inverted immutable index segments. During the search process, Lucene does the search on every segment and merges the results from them. Let's look to the mapping example, it is written in Elixir DSL, but the pure JSON looks pretty the same:

{% highlight elixir %}

    index = [index: "MyIndex", type: "MyType"]

    settings do
      analysis do
        analyzer "standard_snowball",
        [
          filter: ["lowercase", "stop", "snowball"],
          tokenizer: "standard"
        ]
      end
    end

    mappings _source: %{enabled: false} do
      indexes "tags", type: "string", analyzer: "standard_snowball"
      indexes "name", type: "string", analyzer: "standard_snowball"
      indexes "created_time", type: "date"
      indexes "username", type: "string", index: "not_analyzed"
    end

{% endhighlight %}

Here I've defined an index with name "MyIndex" and type "MyType". Also, I've created an analyzer called "standard_snowball", it uses `standard` tokenizer to break sentences into tokens, then it makes them to lowercase with `lowercase` filter, remove stop-words with `stop` filter and stem them with `snowball` filter. I use this analyzer for "tags" and "name" string fields. Important to notice that it will be used both to analyze source data and search queries for these fields. Search query analyzer could be changed at any time, but it's impossible to change analyzer for a source data that is already has been analyzed. Also, it isn't possible to change datatype of fields in most cases. For example, mapping can't be updated to convert a string field to a date field. It's even not possible to rename a field in existing mapping. But if it needs to add a new field, then you can update mapping and the new data will be indexed properly.

To change analyzers of an existed data, all data should be reindexed. In case of `_source` is set to `false`, main data storage should be used for reindexing. If source is stored, then it is possible to create a new index and reindex data from an old index. Usually this process is quite slow, but can be done with a zero-downtime. Of course reindexing isn't an unbeatable challenge, but better to keep in mind that it can create some pain.

Unfortunately ElasticSearch has some disadvantages that can't be easily bypassed when it is used as a storage. First of all, it isn't [ACID](https://en.wikipedia.org/wiki/ACID). It doesn't support transactions, so it isn't possible to rollback first stored document if saving of a second fails.

As I said earlier, it stores documents as JSON, so it is schema less and non-relational. This mean no joining queries, no migrations and other cool features that we love in relationship databases. Yes, ElasticSearch has parent-child relationship, but it comes with certain [restrictions](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-parent-field.html#_parent_child_restrictions). Maybe it's a bit personal, but after more than a year of working with MongoDB I have some reasons to dislike NoSql.

Another problem is a deep pagination. Because ElasticSearch is distributed, it requires a lot of memory to paginate distant pages, it even has limitation by default. If index is distributed between 5 shards and someone need to find first 20 documents, it is possible that all of them may be located on a one shard or be divided between 2-5 shards. To present results, each shard retrieves first 20 documents and pass them to a coordination node, then it sorts 100 documents and return only first 20 of them. If user decided to go to page 1001, system needs to return documents on positions from 20001 to 20020. To make this happen, index should find 20020 documents on each shard, sort 100100 documents in total, and then drop first 100080 of them. To avoid this overkill situation query API has `search_after` parameter, it accepts an id or other unique value from a last document on previous page to retrieve a next page. It's a nice solution to paginate over all data page by page, but it is useless if the user needs to jump on a specific page.

As for security, it is true that old versions of ElasticSearch allow anyone, who can connect to a cluster, do any request, but actual versions have [authorization via xpack](https://www.elastic.co/guide/en/x-pack/current/xpack-security.html) so it's not a problem anymore.

Also, some people complain about robustness. As for me, I've never encountered a situation with `OutOfMemory` errors, so I can't comment this topic. Maybe I don't have that many data.

If you OK with these drawbacks, you can try to use ElasticSearch as a primary database, but I can't recommend doing this. Furthermore, even authors don't promote it as a storage.
