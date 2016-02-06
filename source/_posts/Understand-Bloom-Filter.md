title: Understand Bloom Filter
date: 2016-02-06 10:46:14
tags:
	- Bloom Filter
	- Algorithm
	- White Listing
---

Filter is used on a lot of application with many use cases. While you could use a hash table in a lot of cases, it would usually require same amount of memory capacity of the list you want to filter on, and sometimes it's an issue. Then Bloom Filter comes into the play.

<!-- more -->

# How It Works?
Bloom Filter works by hash each filter string to bits level, and overlap those bits into one large enough bits array. With a simple illustration:
if we convert single char to bits: 
``` a Char to Bits
'a' -> 01100001, 'b' -> 01100010, 'y' -> 01111001, 'c' -> 01100011,
```
and put 'a' and 'b' to a 8 bits array, we have array bf:
```a BloomFilter Bits Array ('a' and 'b')
+---+---+---+---+---+---+---+---+
| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 
+---+---+---+---+---+---+---+---+
| 0 | 1 | 1 | 0 | 0 | 0 | 1 | 1 |
+---+---+---+---+---+---+---+---+
```
Now if we try put Char 'y' into this array, index 3,4,6 do not match so that 'y' is not in the filter list. Nice isn't? Bloom Filter simply works this way.
Now it comes to the bitter side, if we compare 'c' bits with filter array, all elements matched but 'c' is not in the filter list, and that's called **False Positive**. 

# False Positive
Bloom Filter guanrantees return false when the string is not in filter list, but still could return true if it's not in the list.

It's a sacrifice of accuracy of speed and capacity. While that is not perfectly desirable, the outcome influences would be low for some use cases. For example, a large scaled web server might need maintain a blacklist that reject incoming traffics, when response time is crucial, bloom filter would work greatly here even though sometimes it might recognize some OK traffic as blacklisted. (Maybe maintain another trust list that mistakenly filtered out by Bloom Filter?)


# Improve Accuracy and Confidence
False Positive is caused by hash collision. The more overlapping bits filter list string have, the less accuracy your bloom filter could be. 

To decrease the number overlapped bits, first thing you can do is to improve the hash: 256 length of hash string instead of 128 for example.

And of course, you simply can not increase hash lengh forever because length of filer list of not unmanageable, or even length of each filter string is uncontrollable as well. Best hash function is going to lose information from original string thus inevetibly cause collisions.

Another solution I can think of is a **Sharded Bloom Filter**. The idea is, if the filter list is too long, and all the filter hash bits start overlapping all over, a separated filter table grouped by less overlapping bits would scale well. A down side would be more pre-processing before creating the bloom filter, but it usually happens during the initialization of an application so I guess it's not a super big deal.


# Adjust Hash Size and Bits Array Capacity On Demand
While the better hash and larger bits array, the more accuracy you will gain. There should be a point that's "good enough" for your size of data - real world use case huh? 

There are some very standard fancy math that could derive the answer for those number, I couldn't possiblly remember any of them.. Math is hard anyway. One best practice I remember is that: **keep the bits array half empty**. And of course, test frequently your bloom filter with some production data sets!


I just learnt this today, very cool stuff, and that is all I got from Bloom Filter!