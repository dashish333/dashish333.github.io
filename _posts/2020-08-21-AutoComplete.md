---
title: 'ElasticSearch: Driving AutoComplete'
date: 2020-08-21
permalink: /blog-posts/autocomplete/
tags:
  - elastic search
  - autocomplete
  - ngram
  - kibana
  - logstash
---
This is one of the feature delivered by me and my colleagues for an open source project [India Biodiversity portal](https://indiabiodiversity.org/)(ibp). Well, before moving on to highlight the implementation, I must say if you are biodiversity enthusiast then do make a profile [here](https://indiabiodiversity.org/).<br/>
<br/>



While exlporing on the logic to appopriately implement the auto-complete feature as per our requirement,  two concepts are to be taken into account while creating index in the Elastic. These include that one must have appropriate *settings* and *mapping*. Without extending the boring theory any further, let's dive into our core problem statement and how we achieved it. 

# Problem Statement:
So, this whole portal as already mentioned earlier is regarding all about biodiversity, trees, frogs, reptiles, what not! Well this portal is mainly managed by biodiveristy scientist, and hence beside the *common names*  there are *scientific name* associated with each of the observation. So, the feature is about providing autocomplete functionality to end-user, where upon typing a few alphabets the user can see all the possible suggestions.

# Solution:
### Settings:  
<!-- ![ME](/images/mypic.jpg)  -->
```yaml
{"settings": {
    "analysis": {
      "filter": {
        "ngram_filter": {
          "type": "nGram",
          "min_gram": 3,
          "max_gram": 10,
          "token_chars": [
            "letter",
            "digit",
            "symbol",
            "punctuation"
          ]
        }
      },
      "analyzer": {
        "nGram_analyzer": {
          "type": "custom",
          "tokenizer": "whitespace",
          "filter": [
            "lowercase",
            "asciifolding",
            "ngram_filter"
          ]
        },
        "whitespace_analyzer": {
          "type": "custom",
          "tokenizer": "whitespace",
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      },
      "normalizer": {
        "case_insensitive_normalizer": {
          "type": "custom",
          "filter": [
            "asciifolding",
            "lowercase"
          ]
        }
      }
    }
  }
}
```
Once we have the appropriate setting for our index, we succeeded in creating the following mapping, which eventually uses the above definition of *normalizer* , *whitespaceanalyzer*, *ngram analyzer*.
### Mappings:
```yaml
{
  "mappings": {
    "extended_records": {
      "properties": {
        "name": {
          "type": "text",
          "analyzer": "nGram_analyzer",
          "search_analyzer": "whitespace_analyzer",
          "fields": {
            "raw": {
              "type": "keyword",
              "normalizer": "case_insensitive_normalizer"
            }
          }
        },
        "common_names": {
          "properties": {
            "name": {
              "type": "text",
              "analyzer": "nGram_analyzer",
              "search_analyzer": "whitespace_analyzer",
              "fields": {
                "raw": {
                  "type": "keyword",
                  "normalizer": "case_insensitive_normalizer"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

So, once we have an index with the above settings and mapping, the user have the following ease:<br/>
![Common Name AutoComplete](/images/commonName.png)
<br/>
<br/>
and, similary we have something for scientific name:<br/>

![Scientific Name Auto Complete](/images/scientificNameAC.png)

