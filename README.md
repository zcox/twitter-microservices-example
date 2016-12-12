This repository contains code and [slides](https://github.com/zcox/twitter-microservices-example/blob/master/Updating%20Materialized%20Views%20and%20Caches%20Using%20Kafka.pdf) for a [talk at Prairie.Code()](http://prairiecode.amegala.com/sessions/updating-materialized-views-and-caches-using-kafka).

## Building a New Service

Imagine that we are asked to build a new service that provides a single point of read access to data from multiple sources within our company. Data is created and updated by those other sources, not this new service. We may need to query multiple tables or databases, performing complex (and potentially expensive) joins and aggregations. Other services within our company will obtain this derived data from our new service, using it in various ways.

To provide business value, this new service needs to be low latency, providing very fast response times under non-trivial load (e.g. 95th percentile HTTP response time under 5 msec, and max 10 msec, at 1000 HTTP requests/sec). This service may be part of a UI, where [fast responses are important for a good experience](https://www.nngroup.com/articles/response-times-3-important-limits/). Transactional consistency with the other systems of record is not required; we can tolerate data update delays, but they should be bounded (e.g. 95th percentile updates available in this service within 5 sec, and max 10 sec). Breaking these SLAs should result in humans being alerted.

To provide a concrete (but imaginary!) example, let's say we work at Twitter, and this new service provides read access to *user information* comprised of various user profile fields that users can edit, along with summary counts of various things related to the user. This User Information Service can then be used by other services, e.g. front-end desktop and mobile services can quickly get user information to display, the HTTP API can get user information to return as JSON, etc.

![](img/twitter-desktop-profile-highlighted.png)

The User Information Service will provide a single HTTP resource: `GET /users/:userId`, which will either return a `404` response if `userId` does not exist, or a `200` response with user information in a JSON object.

```json
{
  "userId": "...",
  "username": "...",
  "name": "...",
  "description": "...",
  "location": "...",
  "webPageUrl": "...",
  "joinedDate": "...",
  "profileImageUrl": "...",
  "backgroundImageUrl": "...",
  "tweetCount": 123,
  "followingCount": 234,
  "followerCount": 345,
  "likeCount": 456
}
```

Let's assume that the data this service needs is stored in a relational database (e.g. Postgres) in normalized tables. (Twitter's actual data storage is probably not like this, but many existing systems that we're all familiar with do follow this standard model, so let's roll with it.)

- users
  - user_id
  - username
  - name
  - description
  - location
  - web_page_url
  - joined_date
  - profile_image_url
  - background_image_url
- tweets
  - tweet_id
  - text
  - user_id
- follows
  - follow_id
  - follower_id
  - followee_id
- likes
  - like_id
  - user_id
  - tweet_id

A classic implementation of this service would likely end up querying the DB directly using SQL.

```
SELECT * FROM users WHERE user_id = ?

SELECT COUNT(*) FROM tweets WHERE user_id = ?

SELECT COUNT(*) FROM follows WHERE follower_id = ?

SELECT COUNT(*) FROM follows WHERE followee_id = ?

SELECT COUNT(*) FROM likes WHERE user_id = ?
```

Many existing services that generate lots of revenue are implemented like this, and if this approach meets all requirements, then great!

However, many developers have discovered problems with this approach over the years. First off, it's somewhat complex to assemble all of the data needed to fulfill the requests: we are performing multiple queries across multiple tables. This is a fairly simple example; if our service had different requirements, these aggregations could be much more complex, with grouping and filtering clauses, as well as joins across multiple tables.

Second, it's expensive: these queries are aggregating a (potentially) large number of rows. While I may not have a large number of followers, [Katy Perry has over 90 million](https://twitter.com/katyperry/followers). This puts load on the database, increases response latency of our service, and these aggregations are repeated on every request for the same `userId`.

From an architectural perspective, our new service would be sharing data stores with other services. In the world of microservices, this is generally considered an anit-pattern. The shared data store tightly couples the services together, preventing them from evolving independently.

## Introducing a Cache

The standard solution to the above problems is to add a cache, such as Redis. Our service can compute the results of complex queries and store them in the cache, creating a materialized view, which is then easily queried using a simple, fast key lookup. This materialized view cache belongs only to our new service, decoupling it a bit from other services.

However, introducing this cache into our service presents some new problems. While our service can now do a single fast key lookup to get the materialized view from the cache instead of querying the DB, it still has to query the DB and populate the cache on a cache miss, so the complex DB queries remain. When a key is found in the cache, how do we know if the materialized view is up-to-date or stale? We can use a TTL on cache entries to bound staleness, but the lower the TTL, the less effective the cache is.

These problems would be solved if we could just update the materialized views in the cache whenever any data changed in those source tables. Our new service would have no complex DB queries, only simple, fast cache lookups. There would never be cache misses, and the materialized views in the cache would never be stale. How can we accomplish this?

If we have a mechanism to send inserted and updated rows in those tables to Kafka topics, then we can consume those data changes and update the cached materialized views. 

## Sending Data Changes to Kafka Topics

goal: replicate each table into a kafka topic. consumers can then process data changes. each row becomes a message. message key is table's primary key, message value is some serialized form of all columns in the row (prefer avro + schema registry). use log compaction to retain the most recent version of each entity.

- topics
  - users
  - tweets
  - follows
  - likes

Kafka Connect: periodically query DB for new and updated rows in table, convert each row to message and send to kafka topic.

Bottled Water: postgres extension that uses logical decoding to send new, updated and deleted rows to kafka topics. changes get to kafka faster than KC, but may not be production-ready and restricted to postgres.

Dual-Write: services can modify data in DB and then also send data change message to kafka topic. but this adds complexity to service, and need to handle things like not sending msg on failed tx, and could be ordering issues. recommended to extract change out of db after it is committed.

## Updating Materialized Views

user information service receives `GET /users/:userId` request and needs to send back user information JSON object. ideal materialized view is just a simple key lookup by `userId` with all of the fields we need to put into JSON. Could use Redis, but also could just use single table in RDBMS, one column per json field.

users consumer: put user fields from msg into materialized view for `userId` key
  - postgres: INSERT INTO user_information (user_id, username, ...) VALUES (?, ?, ...) ON CONFLICT DO UPDATE
  - redis: HMSET $userId username $username ...

tweets consumer: increment `tweetCount` for `userId`
  - postgres: UPDATE user_information SET tweet_count = tweet_count + 1 WHERE user_id = ?
  - redis: HINCRBY $userId tweetCount 1

follows consumer: increment `followerCount` for `followeeId`, increment `followingCount` for `followerId`

likes consumer: increment `likeCount` for `userId`

## Kafka Streams

In the above, we essentially used an external data store (Postgres or Redis) to join the changelog topics together and store the result of stateful computations (the most recent user fields and counts of tweets/follows/likes by userId). For many use cases, this may be a great solution. However, there are a few potential issues that we may have with this approach. the changelog topics may be so high volume that too many writes are sent to the data store. maybe batching the writes, or scaling up/out the data store is possible, maybe not. maybe the request load on our http service will be so high that too many reads are sent to the data store. maybe we can scale up/out the data store, maybe not. Maybe we don't want to operate/maintain yet another data store. maybe the response time requirements for our service are so low that we cannot tolerate network I/O between our service and the data store.

[Kafka Streams](http://docs.confluent.io/3.1.1/streams/index.html) offers an alternative way to join topics and store state. repartition tweets/follows/likes by userId and then count by key to create tables of (userId, count). join those table changelogs to the user changelog to create user info table (userId, userInformation). this is essentially the same materialized view of user info we stored in the external data store.

State is stored in RocksDB, which is a very fast local in-process key-value store. State is also changelogged to Kafka topics, which can handle a high volume of messages. This can help handle very high volume input changelog topics from other services, if needed. Kafka Streams processing and state is partitioned just like Kafka topics for scaling out. State is fault tolerant because Kafka.

the final user information table can also be output to a changelog kafka topic. this topic could be consumed and simply written to the same external data store we used before (Postgres or Redis). 

one alternative is for each instance of our user information service to consume the user info topic and store it locally in its own RocksDB instance. user info is then obtained from rocksdb to serve the http requests. rocksdb is local and very fast, no network i/o. can scale to match request load by running more/less user info service instances. handling input changelog topic volume and svc request volume is then decoupled. if total user info size exceeds single machine, then partition between svc instances, also need some router in front to route http requests to correct svc instance.

another alternative is to use new Interactive Queries functionality. allows user info state in kafka streams application to be queried directly, without outputting it to some external store (postres, redis, rocksdb). the kafka streams app then becomes the user info svc. may work really well. does couple changelog topic processing with http req serving. also IQ is very new.
