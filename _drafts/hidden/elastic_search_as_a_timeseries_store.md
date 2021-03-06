---
layout: post
title: Elastic Search as as Timeseries data store
---
Hey Baron (and others if you're interested),

You indicated that we sould talk about ElasticSearch after I got back, so this is an attempt to organize my thoughts. We can talk about the details when you have time:

Lucene Internals
Lucene is basically just a well implemented combination of two data structures - a compressed bit array and a trie. The trie holds serves as a fast sorted dictionary of all the tokens encountered in a particular field. The values of this dictionary are the compressed bit arrays which indicate that documents that contain the corresponding token (this is the so-called inverted index). Of these two, we are most interested in the bit array because it has the special property that it can perform boolean operation extremely fast. For our purposes, this means that you can search for all observations from host 42 with name="mysql.queries.q.d9c6a20bb49db88.tput" and Lucene will grab the bit arrays for host and name, quickly AND them together and provide an iterator over all the documents ids that satisfy this query.

Lucene then has to grab the matching documents based upon their ids. So let's take a step back and talk about Lucene data files. Lucene keeps track of plenty of files, but for our purposes it's important to know that most of them either contain the index or the contents of the fields. The index is composed of the term dictionary (which is read into memory as the trie) and the bit arrays (also read into memory). The actual contents of the fields are stored on disk in a column oriented fashion and are memory mapped, so if the data can possibly fit into memory (which might be plausible when scaled to multiple machines), then one need not go to disk at all after the initially loading the data files. There is also a file that contains a list of deleted documents.

Lets consider the next level of abstraction - a Lucene index is composed of several segments which can be thought of as their own individual and independent indices. As new documents are indexed, they are written to a write log. At configurable intervals (or number of updates) the pending documents can be "committed" to a new on-disk segment and the write log is deleted. Documents are only searchable after being committed, so these days, there's also an intermediate step of writing to an in-memory segment so that the documents can become searchable more quickly. Segments are insert-only and docs are written in order that they were received. Unfortunately it's not quite as efficient as simply journaling the data to the disk drive because we have several different files to write. (Though it would be an awesome experiment to see if you could put the data file on a different disk and effectively achieve journaling speeds.)

Updates are a bit of a pain. Since the segments are write only, when you want to update a field in a document all you can really do is retrieve that document, mark it as deleted, and then create a copy of the old document in a new segment with the updated field value. (There have been efforts to make segments rewritable for updating fixed-width values, but I don't think anything's happened here recently.)

Since each segment is an independent index then in order to respond to a query, each index must be searched. Because this takes time segments are periodically merged together. Here's a neat blog post explaining Solr's tiered merge policy. (Complete with cool videos.)

Etc:
I didn't mention it before, but one benefit of using a Trie-based key look up is that it provided extremely fast lookups and on keys according to prefix - so I think a Lucene would be faster with a search like SELECT metric WHERE name LIKE "mysql.queries%" that MySQL would. However I would like your input here as I don't understand MySQL internals very well at this point.
Ignoring the idea of full-text search relevancy, I suspect document sorting works about the same with Lucene as it does with MySQL, though I'm a little fuzzy on the details.
In some sense, Lucene is doing something similar to your 20GBsec Systems blog post. Unlike that post, Lucene first accessing a bit array index of matching documents, but after that Lucene pretty much scans the disk sequentially and uses the index to skip forward past sections that have no matches - but there's not a lot of random access.

What ElasticSearch Adds
Lucene is just the raw tool set for building a search engine. Both Solr and ElasticSearch are "best-practices implementation of Lucene search engines". But both of these add much needed features on top of basic search:
HTTP Access - so that you can retrieve documents remotely
Scalability - Lucene has no notion of multiple machines, so scaling indices beyond a singer server requires a higher level abstraction
Configurability - you have to define the schema of the data, the behavior of search, where data lives on disk, which server index which documents, which which servers service which queries, etc.
Aggregations
ElasticSearch has a very large and at times overwhelming JSON-based configuration API. It allows configuration at all  The search API is also JSON (and perhaps only JSON for now). Scalability is actually where ElasticSearch made it's big splash because it was the first of the two to get distributed search working as a first class feature. You can do really neat things like spin up new indices on the fly, search over multiple indices, drop indices, and manage servers.

It's important to make an aside here -- Montalenti was telling me about Logstash's strategy for managing timeseries data and it might be a pattern worth investigating futher. Basically they create multiple indices named something like env_1-2014.05.28-09:15. And every 15 minutes they create a new index time-stamped with the appropriate time. Wildcards can be used when specifying indices to query against, for instance you can do something like this: 

$ curl http:localhost:9200/env_1-2014.05*/_search?q=host:42

and pull back all matching documents in the month of May. The more recent timestamped indices are stored on big beefy servers so that the most common queries can respond quickly. Older indices are placed on cheaper and slower machines. Also, as the indices age there is apparently a way that small indices can be merged into larger indices. Finally, old indices can be deleted once we no longer need their data.

Finally ES provides a really rich set of aggregation capabilities and new features are being created at a rapid pace. There are two main notions here - the aggregation itself (sum, count, average, min, max, cardinality, standard deviation, etc.) and grouping (they say "buckets"). With ES aggregations you can create and arbitrary nesting of groups and aggregations. For instance, you could perform an ES search for all observations WHERE host=42 AND name LIKE "mysql.queries.%.tput" and then instruct ES to bucket them according to the full name (e.g. mysql.queries.q.d9c6a20bb49db88.tput). For each query bucket you could ask for the average tput. Then inside each bucket you could instruct ES to bucket the timestamps into, for example, 472second intervals and return the average tput for these buckets. Note that this is basically the query you would make to retrieve the first column in our Top Queries dashboard. This data is the sparkline and the rolled up average for tput. Help me think about how we can implement the catchall query.

Etc:
ElasticSearch provides server-side scripting. You could aggregate over the difference between two fields in the same document. Apparently you can use some logic control in the scripting as well - for loops, conditionals.
Despite being a denormalized NoSQL data store, ElasticSearch provides some ability to do JOINs. They do this by allowing for a document to specify its parent as it is indexed. All child documents are routed to the same machine as their parent, thus avoiding a distributed JOIN. We might use this capability to effectively keep the observation, metric, and query tables similar to the way we currently have them set up in MySQL.
Similar to the last bullet, ES allows documents to be indexed with a routing parameter that determines which server the document will be indexed on. This is similar to the way that we're partitioning things in Kafka/Sarama. When queries are specified with that parameter, they are routed to that corresponding server.

Other

Leaning on my experience with Solr, I think there are also some neat tricks that we can do with basic (and fast) text analysis that can make search quite flexible with our current notion of templates (e.g. mysql.queries.{column:abc*|def*}.{key:tput})