---
title: "Postgres ID Generator with Safe JSON Numbers"
date: "2021-06-21T10:33:05+08:00"
description: ""
thumbnail: ""
categories:
  - "database"
tags:
  - "database"
  - "postgres"
  - "snowflake"
# url: relative-url
# aliases:
#   - alias-url-1
# draft: true
# menu: main, side, footer

# theme: mainroad
# thumbnail: images/placeholder.png
# lead: "Lead text"
# comments: true # enable disqus comments for specific page
# authorbox: true # enable authorbox for specifc page
# pager: true # enable pager navigation (prev/next) for specific page
# toc: true # enable Table of Contents for specific page
# mathjax: true # enable MathJax for specific page
# sidebar: "right" # enable sidebar, opts: left, right
# widgets: [ "recent", "categories", "taglist", "social", "languages" ] # enable sidebar widgets in given order
# theme: mainroad
---

Before reading this post, I recommend you to read [A Better ID Generator For PostgreSQL](https://rob.conery.io/2014/05/29/a-better-id-generator-for-postgresql/) written by Rob Conery. To recap briefly, their post had discussed:

1. The problem you will face in the rapid grow of your systems if you used GUID as the primary key.
2. How Twitter generating auto-incrementing keys with [Snowflake](https://github.com/twitter-archive/snowflake).
3. Create a functional Snowflake equivalent for PostgresSQL.

And the third point above will be the focus of discussion in this post.

## The Algorithm

```sql
create schema shard_1;
create sequence shard_1.global_id_sequence;

CREATE OR REPLACE FUNCTION shard_1.id_generator(OUT result bigint) AS $$
DECLARE
    our_epoch bigint := 1314220021721;
    seq_id bigint;
    now_millis bigint;
    -- the id of this DB shard, must be set for each
    -- schema shard you have - you could pass this as a parameter too
    shard_id int := 1;
BEGIN
    SELECT nextval('shard_1.global_id_sequence') % 1024 INTO seq_id;

    SELECT FLOOR(EXTRACT(EPOCH FROM clock_timestamp()) * 1000) INTO now_millis;
    result := (now_millis - our_epoch) << 23;
    result := result | (shard_id << 10);
    result := result | (seq_id);
END;
$$ LANGUAGE PLPGSQL;

select shard_1.id_generator();
```

This algorithm will generate an integer in `64` bits consisting of:

```text
    +----------------+---------------+-------------+
    | Timestamp (41) | Shard ID (13) | Seq ID (10) |
    +----------------+---------------+-------------+
    |<-------------- Unique ID (64) -------------->|
```

| Part      | Bits | Desc.                                                                                            |
| --------- | ---- | ------------------------------------------------------------------------------------------------ |
| Timestamp | 41   | Max valid date should be `timestamp_to_date((2^41 - 1 + 1314220021721)/1000)`, i.e. `2081-04-30` |
| Shard ID  | 13   | Support having `2^13 = 8192` shards                                                              |
| Seq ID    | 10   | Max `2^10 = 1024 ops/ms`                                                                         |

Here we can expand `Seq ID` part to gain the ablility of supporting more operations per ms. But as a trade off, the `Shard ID` will be shrunk.

For example, if we used `10` bits for `Shard ID` and `13` bits for `Seq ID`, we will allow `8192` write operations per ms. Which should be `8M ops/s`. However, **as a reminder**, this `8M ops/s` is **evenly distributed** over 1s. Which means when your system had a writing spike with over 8192 writes in the same millisecond, it can fail.

## Safe Number Problem in JSON

The above algorithm generates a 64-bit integer. In practice, it's not "JSON comaptible". Because in JSON world, _number_ is a "double-like" type. And there's in fact a limitation at JavaScript/ECMAScript level of precision to **53-bit** for integers. The maximum safe integer in JavaScript is `2^53 - 1`. You can check it by running the following JS code in your browser:

```js
Number.MAX_SAFE_INTEGER; // 9007199254740991
Number.isSafeInteger(9007199254740992); // false
```

So, **How we work with 64-bit integers in our API and JavaScript?**

I have two solutions here:

1. Encode a 64-bit integer to text by using binary-to-text encoding methods, e.g. [Base58](https://en.wikipedia.org/wiki/Binary-to-text_encoding#Base58), etc.
2. Generate "safe" integers, which only occupies the lower order 53-bits.

Let's talk about the second solution by tweaking the algorithm above to generate "safe" unique IDs.

## Generate "JSON-Safe" 53-bit Unique IDs for PostgresSQL

| Opt. | Bits (time,shard,seq) | Epoch Offset | Timestamp        | Shard | Seq / TPS    |
| ---- | --------------------- | ------------ | ---------------- | ----- | ------------ |
| 1    | (41, 3, 9)            | 946656000000 | Max `2069-09-06` | 8     | 512 ops/ms   |
| 2    | (41, 4, 8)            | 946656000000 | Max `2069-09-06` | 16    | 256 ops/ms   |
| 3    | (41, 5, 7)            | 946656000000 | Max `2069-09-06` | 32    | 128 ops/ms   |
| 4    | (31, 5, 17)           | 946656000    | Max `2068-01-19` | 32    | 131072 ops/s |
| 5    | (31, 6, 16)           | 946656000    | Max `2068-01-19` | 64    | 65536 ops/s  |
| 6    | (31, 7, 15)           | 946656000    | Max `2068-01-19` | 128   | 32768 ops/s  |
| 7    | (32, 5, 16)           | 0            | Max `2106-02-07` | 32    | 65536 ops/s  |
| 8    | (32, 6, 15)           | 0            | Max `2106-02-07` | 64    | 32768 ops/s  |

Here are some points you need to consider:

1. Use `s` or `ms` for the timestamp?
   - Quick Answer: `ms` for tighter distributions of write operations, and `s` is more flexible to handle write spikes.
2. Set epoch offset or not?
   - Quick Answer: It's a must if using `ms` for timestamp. Otherwise optional. Better not.
3. How many shards should I have?
   - Quick Answer: 16 is sufficient for small and medium applications.
4. TPS considerations?
   - Quick Answer: think of how many write operations your application needs, and the performance (TPS) of your database/shard.

Here's the code for the 7th option:

```sql
CREATE SEQUENCE public.global_id_sequence;
CREATE OR REPLACE FUNCTION public.id_generator(OUT result bigint) AS $$
DECLARE
    now_seconds bigint;
    shard_id int := 1;
    seq_id bigint;
BEGIN
    SELECT nextval('public.global_id_sequence') % 65536 INTO seq_id;

    SELECT FLOOR(EXTRACT(EPOCH FROM CLOCK_TIMESTAMP())) INTO now_seconds;
    result := now_seconds << 21; -- Shard(5) + Seq(16)
    result := result | (shard_id << 16); -- Seq(16)
    result := result | (seq_id);
END;
$$ LANGUAGE PLPGSQL;

SELECT public.id_generator();
```
