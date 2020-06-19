---
title: 'ElasticSearch: Driving User Leaderboard'
date: 2020-06-13
permalink: /blog-posts/leaderboard/
tags:
  - elastic search
  - logstash
  - terms-aggregation
  - bucket sort
  - script
---
Would highly recommend to skim through this [blog](/blog-posts/autocomplete/), as it was my first elastic deliverable for the same project, and this is something more advance version of what I had implemented. Also, I'll really appreciate it.
<br/>
Without further a do we will straight dive into the problem statement.
<br/>
<br/>

## Problem Statement:

Each user on the [ibp](https://indiabiodiversity.org/) portal do some level of activity. These activities and spanned across different type of *module* and *activity* and each activity type is suppose to be subsumed by one of the modules. I hope this wan't a recondite statement. Well, this is not the end, the total score of any user would depend summation of two scores (better call'em score category) which we called *Participation* (P) & *Engagement* (E) score. Each type of activity done by user on the portal is implaced in any one of the above module<br/>
Allow me to help you portray the hierarchy and for that start with imagining a tree with total score (T) as root(level 0), whose immediate child nodes are *P* & *E*. Each of these score category nodes further nodes (@level 3) as a module (M) , finally each of the activity-type(A) will be a child node for these module (M) nodes.
<span style="color: blue;">
After setting the base for the problem, what we were required to do, was to rank users based on value of *T*, unique modules (M), activity type (A), along with returning cumulative count of all activities under each of *M* for each user.
</span>

### Input:
 * *~1.4* *million* documents
 * Each document in an index has following structure:

    ```json
    {
          "module": "Observation",
          "author_id": "abcd",
          "name": "John Doe",
          "root_holder_type": "species.participation.Observation",
          "module_activity_category": "Observation.Organized",
          "profile_pic": "/1ea611a0-7eb1-4ae8-a289-923f463a851c/resources/470.jpg",
          "activity_type": "Posted resource",
          "score_category": "Engagement",
          "@timestamp": "2020-06-15T18:00:10.543Z",
          "created_on": "2012-09-23T00:00:00.000Z",
          "id": 2913,
          "activity_category": "Organized"
      }
    ```

### Output:
 * capability to rank user on -
    * Total Score
    * activity count for each Module 

## Challenge:

Well the first attempt to solve the problem is to look over internet. But eventually I wasn't lucky enough to find a solution. Eventually upon reading other blogs and ES document and reported issues, I found that a direct solution is not available with ES. The solution demands, nested/pipeline aggregation and bucket sort script, which looks somewhat like the below structure (will see actual *kibana* query soon):
- Level 1: aggregate on users
  - Level 2: aggregate on score category
  - Level 2: aggregate on module
    - Level 3: aggregate on activity type
  - Level 2: bucket script to compute total score
  - Level 2: sorting on the desired module count/ total score

Well no doubt the *elastic* has some fantastic capability of searching and aggregating. But our problem was little more complex. We would like to sort first level buckets i.e authors, based on the values from the inner level bucket values or total score computed at level 2. 

<span style="color: Maroon;">
Well this is something for which I couldn't find any solution over web or in ES documentation, as there's no functionality/feature which allows to directly implement a solution for the defined problem. Well as far I can remember this is an open issue with ES team.
Eventually, with after some deliberate efforts, we contrived the ranking feature for our portal by using the ES core functionality itself. It may not be the best solution but served our deliverbale and is time efficient.
<span>

## Solution:
Let's see how the repective kibana query looks like, starting with first level aggregation <br/>

```json
    "group_by_author": {
      "terms": {
        "field": "author_id",
        "size": 11000,
        "min_doc_count": 1,
        "shard_min_doc_count": 0,
        "show_term_doc_count_error": false,
        "order": [
          {
            "_count": "desc"
          },
          {
            "_key": "asc"
          }
        ]
      }
```
I would suggest if you are new to ELK, try executing these queries, it would be a ephemeral fun.
Below is the kibana that we wrote for level 2/3 aggregation along with scripting logic:

```json
{"aggregations": {
        "group_by_score_category_participate":        
         {
          "filter": {
            "term": {
              "score_category.keyword": {
                "value": "Participation",
                "boost": 1
              }
            }
          }
        },
        "group_by_score_category_content": 
        {
          "filter": {
            "term": {
              "score_category.keyword": {
                "value": "Content",
                "boost": 1
              }
            }
          }
        },
        "bucket_by_module": 
        {
          "terms": 
          {
            "field": "module.keyword",
            "size": 100,
            "min_doc_count": 1
          },
          "aggregations": {
            "bucket_by_activity_category": {
              "terms": {
                "field": "activity_category.keyword",
                "size": 100,
                "min_doc_count": 1
              }
            }
          }
        },
        "activity_score": {
          "bucket_script": {
            "buckets_path": {
              "participate": "group_by_score_category_participate>_count",
              "content": "group_by_score_category_content>_count"
            },
            "script": {
              "source": "10*(Math.log10(params.content)+Math.log10(params.participate))",
              "lang": "painless"
            },
            "gap_policy": "skip"
          }
        },
        "activity_score_sort": {
          "bucket_sort": {
            "sort": [
              {
                "activity_score": {
                  "order": "desc"
                }
              }
            ],
            "from": 0,
            "size": 10
          }
        }
      }
}
```
<span style="color: red;">*<span><span style="color: black;">
*size* in above json inside bucket-sort represent the number of top authors that will be returned after inner level bucket sorting. And the *size* which is in the first level aggregation  query, takes into account all the unique aggregated result over authorId.<span>

The final output along with the support from brilliant UI developers-

![leaderboard](/images/leaderboard.png)



