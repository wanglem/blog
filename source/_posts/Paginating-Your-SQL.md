title: Paginating Your SQL - The Hard Way
date: 2015-11-13 01:08:10
tags:
  - mysql
  - pagination
  - api
  - database
  - sql
---
Pagination is one of the key concepts of Restful API. I don't want to repeat its advantage and necessity, because everybody already has the sense to do it. In this article I'm just going to talk about different ways to do it, on MySQL database.

**Enviroment:**
> Macbook Pro OS X El Capitan, i7 with 16G RAM
> MySQL Ver 15.1 Distrib 10.1.8-MariaDB

<!-- more -->
**Disclaimer:** This is not meant to be a design draft for pagination, and I want to be use case specific and sql focused. Practically speaking pagination would also involve a lot application level considerations including data modeling, caching etc.

# A Simple Use Case
Let's say we have a table that stores users' followed topic, the requirement is
> list all "uid", 100 per page

Consider the following schema:
```sql Testing Schema
CREATE TABLE `followed_topics` (
	`id` int(11) UNSIGNED NOT NULL AUTO_INCREMENT,
	`uid` int(11) UNSIGNED NOT NULL,
	`topic_id` int(11) UNSIGNED NOT NULL,
	UNIQUE KEY `uq_user_topic` (`uid`,`topic_id`),
	PRIMARY KEY(`id`)
);
```
I pre-loaded 20M incrementally generated dummy records, 2000 users, each user with 10000 topics.
## First Approach
My instinct tells me the easiest way is something like:
```sql First Approach
SELECT distinct uid 
FROM followed_topics 
ORDER BY uid asc 
LIMIT 0, 100;

SELECT distinct uid 
FROM followed_topics 
ORDER BY uid asc 
LIMIT 100, 100;

...
```
Interestingly, the query time starts staying consistent when pacing towards the later stage. 

| Limit Clause  | Query Time(s) |
| ------------- | ------------- | 
| limit 0, 100; | 0.17 |
| limit 400, 100; | 1.1 |
| limit 800, 100; | 1.94 |
| limit 1200, 100; | 3.48 |
| limit 1600, 100; | 4.51 |


Besides the obvious slowness issue, another major one is we might encounter data loss. For example, just before we load page 2 by running `limit 100, 100`, user "5" suddenly deleted, user "100" would be shifted to page 1, bad bad:
{% img /2015/11/13/Paginating-Your-SQL/data_loss_offset.png 500  "Data Loss from Hard Offsets"%}
## Improved Approach:
```sql Improved
SELECT distinct uid 
FROM followed_topics 
WHERE uid > %%last_seen_uid%% 
ORDER BY uid asc 
LIMIT 100;

...
```
`%%last_seen_uid%%` is the max uid selected on previous page. In this case we don't implicitly rely on other referenced object state: user, topic. Futuremore, the query use table index much better as well, it runs on an average of 200 microseconds, not bad. 

```sql Explain Result 1
admin [tmp]>explain select distinct uid from followed_topics order by uid asc  limit 800, 100;
+------+-------------+-----------------+-------+---------------+----------------+---------+------+----------+-------------+
| id   | select_type | table           | type  | possible_keys | key            | key_len | ref  | rows     | Extra       |
+------+-------------+-----------------+-------+---------------+----------------+---------+------+----------+-------------+
|    1 | SIMPLE      | followed_topics | index | NULL          | uq_user_topic  | 8       | NULL | 19381707 | Using index |
+------+-------------+-----------------+-------+---------------+----------------+---------+------+----------+-------------+
1 row in set (0.00 sec)
```
```sql Explain Result 2
admin [tmp]>explain select distinct uid from followed_topics where uid > 800 order by uid asc limit 100;
+------+-------------+-----------------+-------+----------------+----------------+---------+------+-------+---------------------------------------+
| id   | select_type | table           | type  | possible_keys  | key            | key_len | ref  | rows  | Extra                                 |
+------+-------------+-----------------+-------+----------------+----------------+---------+------+-------+---------------------------------------+
|    1 | SIMPLE      | followed_topics | range | uq_user_topic  | uq_user_topic  | 4       | NULL | 31261 | Using where; Using index for group-by |
+------+-------------+-----------------+-------+----------------+----------------+---------+------+-------+---------------------------------------+
1 row in set (0.00 sec)
```
Above explain shows that the second query using range instead of full table scan (even though on index), and using a lower key_len usage, as a result it scans only on a 30K magnitude compares to 20M from the first one.
Note there is a `Using index for group-by` in Extra column, that is because the query met the **loose index scan** -- uid is on the leftmost of the index (uid, topic_id) we have, plus it's a BTREE istead of HASH, and it enables lookup groups in an index without worrying about all the keys in that index that match the filter condition. Click to see more [MySQL explain](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html) and [Group Optimization](https://dev.mysql.com/doc/refman/5.7/en/group-by-optimization.html).

# A More Modern Use Case
<br \>
> List all topics that followed by a user, newest comes to first, 10 per page.

Looks familiar? Yup, this is likely to be a feed or timeline service use case. Welcome to the modern world. With previous example, we need add a new field --  **created_on**, which is a timestamp field, to the table to do sorting. Consider the following schema:
```sql Updated Testing Schema 
CREATE TABLE `followed_topics` (
	`id` int(11) UNSIGNED NOT NULL AUTO_INCREMENT,
	`uid` int(11) UNSIGNED NOT NULL,
	`topic_id` int(11) UNSIGNED NOT NULL,
	`created_on` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
	UNIQUE KEY `uq_user_topic` (`uid`,`topic_id`),
	KEY `idx_created_on` (`created_on`),
	PRIMARY KEY(`id`)
);
```
## Filter by TIMESTAMP
From our previously improved paging sql, the answer quckly arise:
```sql First Approach
SELECT topic_id 
FROM followed_topics 
WHERE uid = %%current_user%% and created_on > %%last_seen_time%% 
ORDER BY created_on desc 
LIMIT 10;
...
```
Hmm, looks legit.
```sql
admin [tmp]>explain select topic_id from followed_topics where uid = 100 and created_on < "2015-12-20 00:00:00" order by created_on desc limit 10;
+------+-------------+-----------------+------+--------------------------------+----------------+---------+-------+------+-----------------------------+
| id   | select_type | table           | type | possible_keys                  | key            | key_len | ref   | rows | Extra                       |
+------+-------------+-----------------+------+--------------------------------+----------------+---------+-------+------+-----------------------------+
|    1 | SIMPLE      | followed_topics | ref  | uq_follow_topc,idx_created_on  | uq_follow_topc | 4       | const | 1999 | Using where; Using filesort |
+------+-------------+-----------------+------+--------------------------------+----------------+---------+-------+------+-----------------------------+
1 row in set (0.01 sec)
1 row in set (0.00 sec)```

From SQL explain result above, it's not a bad query at all. I'm a little suprised this query is using firesort, because it should sort by created_on which is an indexed column. I guess the reason is the query optimizer would totally ignore the unsed key, even though sort and filter might be two operations.

Note: Interestingly, filesort does not necessarily mean sort on disk, it more of any sort that is not using an index, and essentially by quicksort and mergesort. I guess it's just poorly named. See more in this [**article**](http://s.petrunia.net/blog/?p=24).

Okay, so far so good. If we are working with a high volume of events, however, we could still encounter data loss. The reason is `created_on`, which is the timestamp field, is not unique. Therefore multiple followed topics with same created_on might be trunated from limit clause:
{% img /2015/11/13/Paginating-Your-SQL/data_loss_nonunique_ts.png 500  "Data Loss from Non-unique Timestamp"%}
## Filter by ID for more Uniqueness
How about using PK(id) field? If we rely on the implication that the higher ID the closer created_on, we will have:

```sql Using id Instead of created_on
SELECT topic_id 
FROM followed_topics 
WHERE uid = %%current_user%% and id < %%last_seen_id%% 
ORDER BY id desc 
LIMIT 10;

...
```

In this case we want to use id to filter as well as sorting, apply all filters on same key, which is `uq_follow_topc`, and would not apply further lookups  see below sql explain:
```sql Using index V.S. Using index condition
admin [tmp]>explain SELECT topic_id FROM followed_topics WHERE uid = 100 and id < 1000000 ORDER BY id desc LIMIT 10;
+------+-------------+-----------------+------+------------------------+----------------+---------+-------+------+------------------------------------------+
| id   | select_type | table           | type | possible_keys          | key            | key_len | ref   | rows | Extra                                    |
+------+-------------+-----------------+------+------------------------+----------------+---------+-------+------+------------------------------------------+
|    1 | SIMPLE      | followed_topics | ref  | PRIMARY,uq_follow_topc | uq_follow_topc | 4       | const | 1999 | Using where; Using index; Using filesort |
+------+-------------+-----------------+------+------------------------+----------------+---------+-------+------+------------------------------------------+
1 row in set (0.00 sec)

admin [tmp]>explain SELECT topic_id FROM followed_topics WHERE uid = 100 and id < 1000000 ORDER BY created_on desc LIMIT 10;
+------+-------------+-----------------+------+------------------------+----------------+---------+-------+------+----------------------------------------------------+
| id   | select_type | table           | type | possible_keys          | key            | key_len | ref   | rows | Extra                                              |
+------+-------------+-----------------+------+------------------------+----------------+---------+-------+------+----------------------------------------------------+
|    1 | SIMPLE      | followed_topics | ref  | PRIMARY,uq_follow_topc | uq_follow_topc | 4       | const | 1999 | Using index condition; Using where; Using filesort |
+------+-------------+-----------------+------+------------------------+----------------+---------+-------+------+----------------------------------------------------+
1 row in set (0.00 sec)
```

## Paging Inside a Pagination
In a feed/timeline application, users would be likely to refresh page occasionally trying to pull latest posts, and scrolling all the way down to where they left off. Consider the following scenario:
{% img /2015/11/13/Paginating-Your-SQL/paging_in_pagination.png 300  "Duplicate Computation"%}

As showing above, 1st request pulls down topic 5 - 1, and then topic 11 - 6 are generated; Then 2st request to pull latest 5 topics, which are 11 - 7; Then a 3rd request to continue pulling 6 - 2, with 3 overlapped topics re-computed.

For best resource utilization, we want to implement an exactly-once mechanism by keep tracking of lastest id(`%%latest_seen_id%%`) before each latest pull request. Consider the following sql:
```sql Using latest_seen_id macro
SELECT topic_id 
FROM followed_topics 
WHERE uid = %%current_user%% 
	  and id < %%last_seen_id%% 
	  and id > %%latest_seen_id%%
ORDER BY id desc 
LIMIT 10;
```
Now we should be able to eliminate data re-computation. Back to the previous example, 3rd request now looks like:
```sql 
SELECT topic_id 
FROM followed_topics 
WHERE uid = [user_id]
	  and id < [id_of_topic7]
	  and id > [id_of_topic5]
ORDER BY id desc 
LIMIT 10;
```
