---
layout: post
title:  "How (not) to integrate Elasticsearch testing with RSpec"
date:   2019-03-10 00:47:00 +0200
categories: Dev
---
In this post I want to share my experience in development of an integration
testing tool for a search engine using [RSpec](http://rspec.info) framework,
which is popular among Ruby developers. My goal was to automate generation of
test cases as much as possible. I encountered several problems along the way.
I hope this post will prevent readers from making my mistakes.

I've worked with several companies that use Elasticsearch for data indexing and
searching, however none of them have covered their search requests with tests. I
think the main reason for lack of tests is the complexity of integration
Elasticsearch into a testing environment. The second reason is the enormous
amount of work which is required to support documents stubs for necessary test
cases. The situation with every project has been almost the same: a single
search query is stubbed, then one spec checks that system has sent a request and
received mocked data. This spec has a problem, it checks if a query has been
sent to Elastic, but it does not care about search results.

When I was working in Lobster, we decided to develop a tool which would solve
these problems and allow us to cover search engine responses with specs.
Lobster is a marketplace for user-generated photos and videos. Photographers
connect their accounts from social networks or cloud storages like Instagram,
Flickr, Facebook, Dropbox, etc. to the marketplace, then it fetches their photos
and videos, indexes them and displays to buyers.

### TL;DR
I thought it would be a great idea to develop a tool that records
requests and responses from production as fixtures, sends the same requests
using fixtures in test environment and asserts results.
When an implementation was ready I realised that I was wrong, because I forgot
about TF-IDF model, which is used for scoring in Elastic by default. Don't use this
approach if your product does not allow you to change the scoring model.

### Search engine in UGC marketplace
The main goal for every media marketplace is to make their content searchable.
Nobody would be able buy photos or videos if it isn’t possible to find them,
no matter how beautiful these photos are. In case of social networks people
usually provide a text description for their photos during publishing. However
descriptions are not accurate and they also could be completely unrelated to
objects in photos. The situation with cloud storages is even worse: services
like Dropbox or Google Drive don’t have any descriptions for images at all. The
only related information that is possible to get from cloud storages is EXIF
metadata. It is really helpful if a photo from a cloud storage has latitude and
longitude in EXIF, because with geo-coordinates, at least, it is possible to
query the photo by its location name.

There are several solutions to enhance metadata. First of them is a manual
markup. The quality and accuracy would be ideal, but you need to have thousands
of people who can do it manually. Of course it is impossible to hire so many
people for a small startup. Another way, which is faster and cheaper, but less
accurate, is image recognition with OpenCV and neural networks. With the help of
computer vision algorithms we have gathered names of objects from photos,
dominant colors, emotions, age of people, gender, head position and facial features.

We have been gathering more and more metadata and indexing it in Elasticsearch.
The number of different objects’ features has been increased dramatically since
an ML expert joined the team. As the number of ML features increased, so did the
number of filters and search engine complexity. There were different scoring
rules for different combinations of enabled or disabled filters. Although the
DSL of Elastic is pretty, the search engine code became really complex to
maintain, which sometimes led us to an incorrect ranking. In the end we
decided to write specs for search engine logic in RSpec and run them on a CI
server to be sure that everything works as expected.

![SERP](https://user-images.githubusercontent.com/854386/52372076-7da50980-2a57-11e9-947b-9797c7286677.jpg)


### How to automate testing of a search engine?
Let’s describe a typical test case for a Search Engine Results Page: there are `N` documents in total,
the test makes a query with several filters, parameters and sorting order, then
it asserts that response has only `M` documents, filtered and sorted in
correct order. To test hundreds combinations of filters we need to write and
support hundreds sets of documents and hundreds sets of correct responses. If
somebody changes the search logic, then we have to fix broken sets and correct
responses. For SERP testing it would be almost impossible to maintain stubs for
documents, because even a small change in the logic can affect almost all test
cases.

I gave up on writing specs manually and came up with an idea, which was
excellent at the first glance. I thought that I can make a search query in
production environment and dump results only from the first page, then I would restore the
dump to Elastic in test environment, send the same request there and compare
results. I assumed that documents and their order would be the same, because
I was going to use the same ranking algorithms in both environments.

In other words, I have a search function `F(X) = Y`, it accepts a set of
documents `X` and returns a sorted subset of documents `Y`.
I assumed that if `F(N) = M` then `F(M) = M`, where `N` is the set of all documents in production.

Having made this assumption, I decided to
automate the process of dumping and comparing results. At that point I was
really glad about the simplicity of this idea. My plan was to ask content
moderator for search response samples, which we accept as a valid search
behaviour, dump them and then build testing fixtures. If somebody
breaks search logic, several test cases fail and we figure out a reason. If
changes in logic are expected, then we dump new search results from production
using new search logic and put them into the test environment.

### Development

#### SERP dumper
I started with development of the search dumper: the easiest way to implement
such things in rails is a rake task. It should be run in production and accept
two arguments: array of search requests' ids which are approved by content
moderator – `search_request_ids`, number of documents to dump from each response
– `number_of_items`. Search dumper generates `search_dump.json` file that
contains current configurations of the search engine in `search_config` field,
also it contains `data` payload with all request samples. Each request's
sample includes filters and parameters in `request` field, sorted document IDs
with ranking scores inside the `sorted_results_ids` field and raw documents in
`contents` field.
![dumper-schema](https://user-images.githubusercontent.com/854386/53920262-d07cdb80-406c-11e9-85e0-c372daa3199f.jpg)
I have planned to use obtained dumps as fixtures, to compare document order
from requests in test environment with results that have been acquired in
production.

I am not going to show a sample of dumper's code here, because dumper is tightly
coupled with classes and libraries which are used to interact with Elastic, so
there is no sense in doing it. The logic behind it is quite simple anyway: send
requests, serialize responses to json, write data to a file.

#### Setting up Elasticsearch in a test environment
We need to run Elastic in a test environment to restore the dump and send
queries. We also need to clean it after each run of a test suit. Setting it up
was easier than I expected, because Elastic supports in-memory cluster which
simplifies this task. There is no need to care about data cleaning if new and
clean isolated cluster starts in-memory for each test suit. This approach also
prevents you from corrupting local data, which could happen in case
you decided to use the same cluster that is used in development, but with a
separate namespace for testing. There is even an [elasticsearch-extensions
gem](https://github.com/elastic/elasticsearch-ruby/tree/master/elasticsearch-extensions#testcluster)
that does all dirty work with managing in-memory clusters.

Here is how to run and stop in-memory cluster with RSpec, using
elasticsearch-extensions gem:
```ruby
# spec/spec_helper.rb

# start an in-memory cluster for Elasticsearch as needed
config.before :all, elasticsearch: true do
  unless Elasticsearch::Extensions::Test::Cluster.running?(on: 9250)
    Elasticsearch::Extensions::Test::Cluster.start(port: 9250, nodes: 1, timeout: 120)
  end
end

# stop elasticsearch cluster after test run
config.after :suite do
  if Elasticsearch::Extensions::Test::Cluster.running?(on: 9250)
    Elasticsearch::Extensions::Test::Cluster.stop(port: 9250, nodes: 1)
  end
end
```

I don’t want to go deep into in-memory cluster configuration in this post. For
those who are interested, here is a great
[article](https://medium.com/@rowanoulton/testing-elasticsearch-in-rails-22a3296d989)
on this topic.

#### Specs generation
When I was satisfied with in-memory cluster, I decided to develop a generator
for specs. First of all, the generator should deserialize `search_dump.json`
file and create a search configuration in the database. The configuration is a
record that stores information about boosts and fields which are used in
different queries. For each search case, generator should create a new index in
the Elastic cluster. After that it should format dumped documents into a bulk
and index the bulk in Elastic. Bulk indexing save lots of time if you need to
index many documents. Interaction with Elastic goes through HTTP API, it takes
time to open a new connection for each request. It makes sense to send as few
requests as possible. For example, instead of sending 1000 requests to index
1000 documents, you can send only one bulk request with all documents.

Elastic is a near real time storage, it means that in our case next search
request could return 0 results, because Elastic didn't have time to refresh
index before the query. It’s important to force-refresh index of a cluster when
documents are indexed. When index is refreshed we need to send a query from the
test sample and assert if the sorting order from results is the same as in the
sample.

I know that there are many ways to interact with Elastic from ruby, search logic
could also be encapsulated in different ways. I want to show the general idea behind
the specs generator in RSpec, so I will use abstract class names. Lets assume
that your search logic is implemented in `SearchClass`. For this example I use a
repository pattern from official [elasticsearch-rails
gem](https://github.com/elastic/elasticsearch-rails) for ruby to interact with
Elasticsearch.
```ruby
# spec/services/search_class_spec.rb
require 'rails_helper'
DUMP_NAME = 'search_dump.json'

RSpec.describe SearchClass, elasticsearch: true do
  let(:repository) { ElasticRepository.new }

  # create index for a test
  before :each do
    begin
      repository.create_index!(number_of_shards: 1)
      repository.refresh_index!
    rescue => Elasticsearch::Transport::Transport::Errors::NotFound
    end
  end

  # delete index after a test
  after :each do
    begin
      repository.delete_index!
    rescue => Elasticsearch::Transport::Transport::Errors::NotFound
    end
  end

  # load dump
  file = File.join(Rails.root, 'spec', 'fixtures', DUMP_NAME)
  dump = JSON.parse(File.open(file, 'r').read, symbolize_names: true)

  # iterate over test samples
  dump[:data].each do |sample|
    it "compares search results #{sample[:request][:id]}" do
      # load configurations for a search engine
      SearchConfiguration.create!(dump[:search_config])

      # build a batch for indexing
      batch = sample[:contents].map do |record|
        {
          index: {
            _id: record[:document_id],
            data: record
          }
        }
      end

      # use bulk indexing API to index the batch of documents
      repository.client.bulk(
        index: repository.index,
        type: repository.type,
        body: batch
      )
      repository.refresh_index!

      # build a search request
      request = SampleSearchRequest.new(sample[:request])

      # send search request
      results = described_class.new.perform(request)

      # compare results
      expect(results[:data].map(&:id)).to eq(sample[:sorted_results_ids])
    end
  end
end
```

When I tried to launch the spec, it didn't work properly: order of documents was
completely messed up.

### What can go wrong?
Let's get back to the initial assumption. I thought that I could make a query
across `N` documents and get a sorted subset of documents `M` with the search function
`F(N) = M`. Then I was going to create a new index with these `M` documents and
query it with the same search function `F`. I expected to have `F(M) = M`, because it
seems logical, but in practice I got `F(M) = M'`, where `M'` includes the same
documents as the subset `M`, but sorted in a different order. The root of this mistake
is the scoring model that is used in Elasticsearch by default.

Elastic makes several steps to process a search query. First of all it filters
documents. Filtering is cost effective, because it just determines if documents satisfy
conditions from the query. Then Elastic applies scoring queries to
filtered document set. Scoring defines weights for each document. In the end
Elastic sorts documents by their scores and other fields that were set for the
query. ![search query
schema](https://user-images.githubusercontent.com/854386/53922975-cbbd2500-4076-11e9-8480-6a43ae781821.jpg)

To score documents Elasticsearch use
[TF-IDF](https://en.wikipedia.org/wiki/Tf–idf): Term Frequency – Inverse
Document Frequency. It is a numerical statistic that is intended to reflect how
important a word is to a document in a collection. Term Frequency – the number
of times that a term occurs in the document. Inverse Document Frequency – the
logarithm of the total number of documents in the index, divided by the number
of documents that contain the term.

### Full-text search and TF-IDF
Before going further, it’s important to understand why full-text search uses
IDF. It shows how often the term appears among all documents in the
collection. The more often, the lower is the weight of the term. Common terms
like “and” or “the” contribute little to relevance, as they appear in most
documents, while uncommon terms like “elastic” or “capybara” help to zoom in on
the most interesting documents.

By default Elastic uses [Okapi BM25](https://en.wikipedia.org/wiki/Okapi_BM25) scoring model. It is based on TF-IDF and also
provides field normalization. Fields normalization adds extra precision for
scoring. If a term has been found in a field that contains 5 words, then
document that contains this field is more important than another document that
has a field with 10 words and the same term among them. 

Let's return to our problem: we have several million documents in production,
but a test sample contains only `M` documents. Despite sending the same
query, there would be different Inverse Document Frequency values for documents
in production and for small dumped document sets. Different IDF values lead us
to different scores for the same documents. Sorting order for documents from the
sample wouldn't be the same as genuine order dumped from production.
My initial assumption was wrong since I did not account for IDF in the scoring model.

### The solution
At this point I was extremely upset and angry, because I completely forgot about
TF-IDF when I was designing tools for integration Elasticsearch with RSpec.
However, instead of rejecting the broken idea and making something that would
work in a completely different way, we decided that it is perfect time to change
the logic of our search engine.

Previously we had lots of cases when someone who has searched for photos came
to the development team and said something like, “Our ranking is broken! I've
searched for `cute dog`, now look at this beautiful picture of a dog, it has
both tags: `cute` and `dog`. It should be higher in results than another
picture, which has only the `dog` tag from my query. It means that first photo is
more relevant to my query and should be higher in results." In these cases I
usually make an [explain
query](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-explain.html)
to Elastic and find out that the picture with tags `dog` and `cute` also
contains 50 more unrelated tags. People tend to add trash tags to their photos
in instagram, because they want to be discovered and get some likes. As for the
photo, which was higher in results, it had only two tags: `dog` and `puppy`. As
I wrote earlier, Okapi BM25 model uses fields normalization, it treats the
document with 2 matched terms among 50 as less important than the document
with 1 matched term among 2. Because of the fields normalization, the photo with
1 matched tag had higher score than the photo with two matched tags.

Actually this normalization makes sense, it's like a built-in protection from
tags spammers who try to cheat search algorithm and push their photos to as
many queries as possible. On the other hand, we are selling photos, we are not
making a text search engine. For people who want to buy a photo the most
important thing is the fact that it includes objects from their search request.
When people search for `cute dog`, they
don't really care about other 48 tags if the photo is nice and contains a dog
which is cute. Besides, IDF also ruins people's expectations. They want to
see photos with desired objects and don't care about the frequency of the tags
among all photos in the database.

The second argument was more persuasive and important for us. Considering this,
we decided to change ranking model in the product and disabled Okapi
BM25. Even before the whole story with specs, we already thought that TF-IDF is
harmful for us in most scenarios. By disabling it, we not only fixed our testing
system, but also made the search engine better from the perspective of media
marketplace. People buy photos, not tags or texts.

We switched from Okapi BM25 model to
[`constant_score`](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/query-dsl-constant-score-query.html).
In this case Elastic evaluates score for a document as a sum of preset scores
for each matched field of the document. Since fields normalization isn't used
anymore, it makes sense to [disable
norms](https://www.elastic.co/guide/en/elasticsearch/reference/current/norms.html)
in mappings with `norms: false`. This setting saves some disk space for the
index.

Another important thing to mention is the implementation of results comparator
for specs. Since TF-IDF is disabled, scores deviation almost disappears. When
TF-IDF is enabled, documents usually have different float values for scores, but
without it there would be many documents with the same scores. It means that
Elastic may return documents with the same score in different order for the same
query, which is sent several times. To solve this problem, it is possible to add
another field for a sorting strategy besides score; for example, date of creation.
In this case, subsets of documents with the same score would be sorted
by date of creation.

### Continuous Integration
After switching from Okapi BM25 to `constant_score` specs changed to green, but
it's not the end of the story. Previously I mentioned that another goal for this
task was to run specs in a CI service. In theory it was easy, but in practice I got
a problem with custom tokenizers. We use synonym tokenizers for several search
features. Each tokenizer has its own set of synonyms, some of them are extremely
large. There are two ways to define synonym sets for tokenizers. First of them
is defining synonyms in mappings. However if your set is too big, it’s
recommended to define it in a file, otherwise mappings would be bloated. Here
comes the problem: we had plenty of files with synonyms, but since version 5,
Elasticsearch requires a file's path to be relative to the directory with its
configurations. It's implemented this way to limit access of Elastic to other
directories outside of its own directory. For development, it's better to store
files with synonyms in repositories and symlink them to a directory with Elastic
configurations. However it's not possible do something like that for CI. At that
time we were using VexorCI, their support is amazing! We have provided them
files with synonyms and they have build a special image of Elasticsearch for us,
it was bundled with synonyms files. After that we were able to run specs for
Elastic in CI service.

### Conclusion
I strongly advise not to use this approach, because disabling TF-IDF isn't what
you usually do with a full-text search engine. I had been lucky that
we were making a product which sells photos, otherwise I would have thrown the
idea away.
