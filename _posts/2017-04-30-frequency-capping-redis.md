---
layout: post
title: Implementing Frequency Capping with Redis
categories: redis
date:   2017-04-30 00:18:23 
---


Frequency capping is a very important concept in digital advertising. Simply put, frequency capping is a strict upper bound for an ad that can be shown to a specific user over a period of time. The period of time can be daily (24 hour period), weekly or monthly. For instance, think of an ad with its daily cap set to 5 and monthly cap set to 20. This means that a user with id user_id1 should not see this particular ad more than 5 times a day and 20 times a month. For our use case, we will consider daily and monthly capping.

Whenever an impression is generated, the ad-server receives the impression request, which is a simple HTTP request with the user id stored in a cookie. To ensure that the user hasn't exceeded the capping limit, the ad server must compute the number of times the ad has been shown to this user on a daily, weekly and monthly basis for every request. Imagine that for a second. If an ad server gets 10000 req per second, it must calculate the daily and monthly count for this ad to this user <em> ON EVERY REQUEST 
</em>. Fortunately, we have the right candidate for this sort of job. Redis!

Redis is phenomenal. It is stunningly fast, you can set it up quickly and it's easy to build a prototype. 

This is what I look like when I think of Redis
![zen](https://cloud.githubusercontent.com/assets/7692552/25563041/d22fc026-2db0-11e7-9757-78c4d4d498f6.jpeg)

There are are a couple of ways

## Hashes

This would be the fastest way to do it. 

Further breaking down our problem, all we need to do is maintain a impression count for a user on a daily, weekly and monthly basis for a campaign. Assume that campaign id is 1 and user id is 1234. 

Our Redis keyspace contains one Hash, with the name `user_impression_counts`. Our Hash fields have the following template:

```
daily_count:user:<user_id>:campaign:<campaign_id>:date:<date_stamp>
monthly_count:user:<user_id>:campaign:<campaign_id>:month:<month_stamp>
```

`<date_stamp>` has format `yyyyMMdd` and `<month_stamp>` has format `yyyyMM`. So for example, if the date is 2nd August, which is the 8th month of the year, then `date_stamp` = `20170802` and `month_stamp` = `201708` .

Here is the python code to get the daily and monthly count.

```python
USER_CAMPAIGN_IMPRESSION_COUNT_HASH_NAME = "user_impression_counts"
USER_CAMPAIGN_DAILY_IMPRESSION_COUNT_HASH_KEY = "daily_count:user:<user_id>:campaign:<campaign_id>:date:<date_stamp>"
USER_CAMPAIGN_MONTHLY_IMPRESSION_COUNT_HASH_KEY = "monthly_count:user:<user_id>:campaign:<campaign_id>:date:<month_stamp>"

def get_daily_impression_count_for_user(campaign_id,user_id,date):
    # date is of format yyyyMMdd.
    # campaignId is an integer. Convert it to a String
    key = USER_CAMPAIGN_DAILY_IMPRESSION_COUNT_HASH_KEY.replace("<campaign_id>",str(campaign_id)).replace("<user_id>",user_id).replace("<date_stamp>",date)
    count_str = redis_db.hget(USER_CAMPAIGN_IMPRESSION_COUNT_HASH_NAME,key)
    if count_str is None:
        count_str = 0
    return int(count_str)

def get_monthy_impression_count_for_user(campaign_id,user_id,month_stamp):
    # month is of format yyyyMM
    key = USER_CAMPAIGN_MONTHLY_IMPRESSION_COUNT_HASH_KEY.replace("<campaign_id>",campaign_id).replace("<user_id>",user_id).replace("<month_stamp>",month_stamp)
    count_str = redis_db.hget(USER_CAMPAIGN_IMPRESSION_COUNT_HASH_NAME,key)
    if count_str is None:
        count_str = 0
    return int(count_str)

```


Now that we have the daily and monthly impressions served to this user, we can easily compare these counts against the capping limit set for our campaign in our CMS. If we haven't exceeded our capping limit, the user would be shown the ad and a impression would be generated.

Next, let's look at how to increment our counts when we get an impression.

```python

# We need not worry about race conditions, as Redis guarantees increments to be an Atomic operation.

def increment_daily_impression_count_for_user(campaign_id,user_id,date):
    key = USER_CAMPAIGN_DAILY_IMPRESSION_COUNT_HASH_KEY.replace("<campaign_id>",str(campaign_id)).replace("<user_id>",user_id).replace("<date_stamp>",date)
    # date is of format yyyyMMdd.
    # We just increment the count for the passed date by 1 .
    redis_db.hincrby(USER_CAMPAIGN_IMPRESSION_COUNT_HASH_NAME, key, 1)

def incrememnt_monthly_impression_count_for_user(campaign_id,user_id,month_stamp):
    # month is of format yyyyMM
    key = USER_CAMPAIGN_MONTHLY_IMPRESSION_COUNT_HASH_KEY.replace("<campaign_id>",str(campaign_id)).replace("<user_id>",user_id).replace("<month_stamp>",month_stamp)
    # We just increment by 1
    redis_db.hincrby(USER_CAMPAIGN_IMPRESSION_COUNT_HASH_NAME, key, 1) 

```


We're done. This is by far the fastest way to do it.

## Sorted Set

A sorted set in Redis is a Set with a key , a score and a value. The set is ordered on the basis of the 'score' of an individual 'member'. For our use case, we will build fat Sorted sets with name `impression:user:<user_id>:campaign:<camp_id>`. The uniqueness of elements in a set depends on the 'value'. For the members in a Sorted Set to be distinct, their 'values' need to be distinct.

So, we need to come up with some random alphanumeric string that we can set as value. We can use a `UUID` for this purpose.

Here is how we would put a record into the sorted set:

```python

import uuid
import datetime

USER_CAMPAIGN_IMPRESSION_SORTED_SET = 'impression:user:<user_id>:campaign:<campaign_id>'

def put_impression_entry_to_sorted_set(campaign_id,user_id,datetime):
    # format of date time : yyyy-MM-dd HH:mm:ss
    score = get_timestamp(datetime)
    value = uuid.uuid4()
    redis_db.zadd(USER_CAMPAIGN_IMPRESSION_SORTED_SET.replace("<campaign_id>", str(campaign_id)).replace("<user_id>",user_id)
                  ,value
                  ,score
                  )


def get_timestamp(datetime):
    return int(datetime.datetime.strptime(datetime, '%Y-%m-%d %H:%m:%s'))

```

This takes care of building the Sorted Set. Now, let's look into how we can get the impression counts.


```python

def get_daily_impression_count_for_user(campaign_id,user_id,date):
    date_obj = datetime.datetime.strptime(date,"%Y%m%d")
    start_of_date = date_obj + datetime.timedelta(seconds=1)
    end_of_date = date_obj + datetime.timedelta(hours=23,minutes=59,seconds=59)
    start_of_date_timestamp = start_of_date.timestamp()
    end_of_date_timestamp = end_of_date.timestamp()
    key = USER_CAMPAIGN_IMPRESSION_SORTED_SET.replace("<campaign_id>",str(campaign_id)).replace("<user_id>",user_id)
    #The count for the entire day
    count_str = redis_db.zcount(key,start_of_date_timestamp,end_of_date_timestamp)
    if count_str is None:
        count_str = 0
    return int(count_str)

```

What exactly did we just do? Well, for any impression that is generated by the system, there is a `timestamp` (think python's datetime) object attached to it. To put this impression into our sorted set, we get the timestamp in milliseconds from our datetime object and use that as score.
Next, when we wish to know the 'daily' count for the impression, i.e on a particular date, say 30th April 2017 (2017-04-30), we build two timestamps. The first is the start of this date, i.e `2017-04-30 00:00:01` (the very first second that day), and the last is the end of the day '2017-04-30 23:59:59'. We use Redis's `ZCOUNT` function to count the number of impressions between these two timestamps.
The same logic would go to get the monthly impression count for user. 

I would recommend going with Redis Hashes. `ZCOUNT` is a `O(N)` operation, whereas Hashes is a `O(1)` operation. 


