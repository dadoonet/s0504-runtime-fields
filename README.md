# Elastic Daily Bytes S05E04: Runtime fields

## Setup

Open:

* <https://elastic-daily-bytes-s05.kb.us-central1.gcp.cloud.es.io:9243/app/dev_tools#/console>
* <https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/ZonedDateTime.html>

## Introduction

So far, we have seen the Search API, text analysis and some text queries which are important to know.

But what if you would like to query for some content which is not available as you would like?

For example, let say I would like to search posts that have been created over the week-end.

```json
GET /bytes-discuss-02/_search
```

As we can see, I don't have such a `day_of_week` field. If I want to get such a field, I'd need to reindex my whole dataset... But I'm not sure yet about the rules to implement.

## Schema on Read vs Schema on Write

Elasticsearch is known to be a schema on write search engine. Which means that you must define everything and index your data to be able to retrieve what you are searching for.

But do you know that Elasticsearch is also now a Schema on Read based search engine? That means that you can build your schema on the fly, while reading the dataset.

The name of this feature is "runtime fields".

## Runtime fields - basic

### Add a `day_of_week` field

Let's create a basic example. We just want to emit a fake value for now. The type of the field is mandatory as this will act as a "normal" field for Elasticsearch.

```json
GET /bytes-discuss-02/_search
{
  "runtime_mappings": {
    "question.day_of_week": {
      "type": "keyword",
      "script": {
        "source": """
        emit('Monday');
        """
      }
    }
  }
}
```

### Show it. It's not in `_source`

When we run it, we can't see it's value. Of course, it's not part of the `_source` field but we can retrieve it using the `fields` parameter:

```json
GET /bytes-discuss-02/_search
{
  "runtime_mappings": {
    "question.day_of_week": {
      "type": "keyword",
      "script": {
        "source": """
        emit('Monday');
        """
      }
    }
  },
  "fields": [
    "title",
    "question.day_of_week"
  ],
  "_source": false
}
```

## Compute the day of the week

### What is this Object?

`Debug.explain` is an helpful method to understand what type of objects we are working on.

Replace `emit('Monday');` by `Debug.explain(doc['question.date'].value);`:

```json
GET /bytes-discuss-02/_search
{
  "runtime_mappings": {
    "question.day_of_week": {
      "type": "keyword",
      "script": {
        "source": """
        Debug.explain(doc['question.date'].value);
        """
      }
    }
  },
  "fields": [
    "title",
    "question.day_of_week"
  ],
  "_source": false
}
```

This is giving us a Java classname `ZonedDateTime`. We can open the [JavaDoc](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/ZonedDateTime.html).

### Look at the `dayOfWeek`

In the documentation we can see the `dayOfWeek` field. Let's access it:

```json
GET /bytes-discuss-02/_search
{
  "runtime_mappings": {
    "question.day_of_week": {
      "type": "keyword",
      "script": {
        "source": """
        Debug.explain(doc['question.date'].value.dayOfWeek);
        """
      }
    }
  },
  "fields": [
    "title",
    "question.day_of_week"
  ],
  "_source": false
}
```

### display the name

Open the [JavaDoc for DayOfWeek](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/DayOfWeek.html). It has a `getDisplayName()` method. Let use it with `TextStyle.FULL` and `Locale.ROOT` parameters:

```json
GET /bytes-discuss-02/_search
{
  "runtime_mappings": {
    "question.day_of_week": {
      "type": "keyword",
      "script": {
        "source": """
        Debug.explain(doc['question.date'].value.dayOfWeek.getDisplayName(TextStyle.FULL, Locale.ROOT));
        """
      }
    }
  },
  "fields": [
    "title",
    "question.day_of_week"
  ],
  "_source": false
}
```

This gives the value we are expecting as a `String`.

### Emit the name

We just have to replace `Debug.explain` by `emit` as we saw earlier:

```json
GET /bytes-discuss-02/_search
{
  "runtime_mappings": {
    "question.day_of_week": {
      "type": "keyword",
      "script": {
        "source": """
        emit(doc['question.date'].value.dayOfWeek.getDisplayName(TextStyle.FULL, Locale.ROOT));
        """
      }
    }
  },
  "fields": [
    "title",
    "question.day_of_week"
  ],
  "_source": false
}
```

## Emit also the name for the answer

We can add the same thing for the `solution`:

```json
GET /bytes-discuss-02/_search
{
  "runtime_mappings": {
    "question.day_of_week": {
      "type": "keyword",
      "script": {
        "source": """
        emit(doc['question.date'].value.dayOfWeek.getDisplayName(TextStyle.FULL, Locale.ROOT));
        """
      }
    },
    "solution.day_of_week": {
      "type": "keyword",
      "script": {
        "source": """
        emit(doc['solution.date'].value.dayOfWeek.getDisplayName(TextStyle.FULL, Locale.ROOT));
        """
      }
    }
  },
  "fields": [
    "title",
    "question.day_of_week",
    "solution.day_of_week"
  ],
  "_source": false
}
```

## Search for questions asked/answered over the week-end (Sat-Sun)

It's a "normal" field we can use to query. We saw the previous days what an analyzer is and what is a `multi_match` query. So let's apply this today:

```json
GET /bytes-discuss-02/_search
{
  "runtime_mappings": {
    "question.day_of_week": {
      "type": "keyword",
      "script": {
        "source": """
        emit(doc['question.date'].value.dayOfWeek.getDisplayName(TextStyle.FULL, Locale.ROOT));
        """
      }
    },
    "solution.day_of_week": {
      "type": "keyword",
      "script": {
        "source": """
        emit(doc['solution.date'].value.dayOfWeek.getDisplayName(TextStyle.FULL, Locale.ROOT));
        """
      }
    }
  },
  "fields": [
    "title",
    "question.author.username",
    "solution.author.username",
    "question.day_of_week",
    "solution.day_of_week"
  ],
  "query": {
    "multi_match": {
      "analyzer": "whitespace",
      "query": "Saturday Sunday",
      "fields": [
        "*.day_of_week"
      ]
    }
  }, 
  "_source": false
}
```

It works! We now know how to search on discuss topics that have been posted or answered over the week-end.

## Response time

But look at the response time. It's not super fast if we compare with a regular search...

```json
GET /bytes-discuss-02/_search 
{
  "query": {
    "match": {
      "question.text": "Elasticsearch"
    }
  }
}
```

### Add the runtime field to the mapping

We can "just" create a new index with the runtime field we created earlier:

```json
DELETE bytes-discuss-04
PUT bytes-discuss-04
{
  "settings": {
    "index": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    },
    "analysis": {
      "filter": {
        "english_stemmer": {
          "type": "stemmer",
          "language": "english"
        }
      },
      "tokenizer": {
        "char_group": {
          "type": "char_group",
          "tokenize_on_chars": [
            "whitespace",
            "punctuation"
          ]
        }
      }, 
      "analyzer": {
        "html_analyzer": {
          "filter": [ "lowercase" ],
          "char_filter": [ "html_strip" ],
          "tokenizer": "standard"
        },
        "title_analyzer": {
          "char_filter": [ "html_strip" ],
          "tokenizer": "char_group",
          "filter": [ "lowercase", "english_stemmer" ]
        },
        "path_analyzer": {
          "filter": [ "lowercase" ],
          "tokenizer": "path_hierarchy"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "category_name": {
        "type": "keyword"
      },
      "duration": {
        "type": "unsigned_long"
      },
      "question": {
        "properties": {
          "author": {
            "properties": {
              "avatar_template": {
                "type": "text",
                "analyzer": "path_analyzer"
              },
              "name": {
                "type": "text",
                "fields": {
                  "keyword": {
                    "type": "keyword"
                  }
                }
              },
              "username": {
                "type": "search_as_you_type"
              }
            }
          },
          "date": {
            "type": "date"
          },
          "reads": {
            "type": "short"
          },
          "text": {
            "type": "text",
            "analyzer": "html_analyzer"
          },
          "day_of_week": {
            "type": "keyword",
            "script": {
              "source": """
              emit(doc['question.date'].value.dayOfWeek.getDisplayName(TextStyle.FULL, Locale.ROOT));
              """
            }
          }
        }
      },
      "solution": {
        "properties": {
          "author": {
            "properties": {
              "avatar_template": {
                "type": "text",
                "analyzer": "path_analyzer"
              },
              "name": {
                "type": "text",
                "fields": {
                  "keyword": {
                    "type": "keyword"
                  }
                }
              },
              "username": {
                "type": "search_as_you_type"
              }
            }
          },
          "date": {
            "type": "date"
          },
          "post_number": {
            "type": "short"
          },
          "reads": {
            "type": "short"
          },
          "text": {
            "type": "text",
            "analyzer": "html_analyzer"
          },
          "day_of_week": {
            "type": "keyword",
            "script": {
              "source": """
              emit(doc['solution.date'].value.dayOfWeek.getDisplayName(TextStyle.FULL, Locale.ROOT));
              """
            }
          }
        }
      },
      "title": {
        "type": "text",
        "analyzer": "title_analyzer"
      },
      "topic": {
        "type": "unsigned_long"
      }
    }
  }
}
```

### Reindex

And reindex our dataset into this new index:

```json
POST _reindex?refresh=true
{
  "source": {
    "index": "bytes-discuss-02"
  },
  "dest": {
    "index": "bytes-discuss-04"
  }
}
```

Automatically, Elasticsearch is going to take the full advantage of the mapping and build the data structure for the runtime fields.

So if we search again for questions asked/answered over the week-end (Sat-Sun) but on this new index:

```json
GET /bytes-discuss-04/_search
{
  "fields": [
    "title",
    "question.author.username",
    "solution.author.username",
    "question.day_of_week",
    "solution.day_of_week"
  ],
  "query": {
    "multi_match": {
      "analyzer": "whitespace",
      "query": "Saturday Sunday",
      "fields": [
        "*.day_of_week"
      ]
    }
  }, 
  "_source": false
}
```

## Lookups (like a join?)

Let say we have an index containing a list of categories:

```json
DELETE bytes-discuss-categories
PUT bytes-discuss-categories
{
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword"
      },
      "description": {
        "type": "text"
      }
    }
  }
}
POST bytes-discuss-categories/_bulk
{ "index": { }}
{ "name": "Elasticsearch", "description": "Elasticsearch is a distributed, RESTful search and analytics engine capable of addressing a growing number of use cases. As the heart of the Elastic Stack, it centrally stores your data for lightning fast search, fine‑tuned relevancy, and powerful analytics that scale with ease." }
{ "index": { }}
{ "name": "Kibana", "description": "Kibana is a free and open user interface that lets you visualize your Elasticsearch data and navigate the Elastic Stack. Do anything from tracking query load to understanding the way requests flow through your apps." }
{ "index": { }}
{ "name": "Logstash", "description": "Logstash is a free and open server-side data processing pipeline that ingests data from a multitude of sources, transforms it, and then sends it to your favorite \"stash.\"" }
{ "index": { }}
{ "name": "Beats", "description": "Beats is a free and open platform for single-purpose data shippers. They send data from hundreds or thousands of machines and systems to Logstash or Elasticsearch." }
```

We can enrich our response object with the content of this index by doing a lookup on the `category_name`:

```json
GET bytes-discuss-04/_search
{
  "runtime_mappings": {
    "category_description": {
        "type": "lookup", 
        "target_index": "bytes-discuss-categories", 
        "input_field": "category_name", 
        "target_field": "name", 
        "fetch_fields": [ "description" ]
    }
  },
  "fields": [
    "title",
    "category_name",
    "category_description"
  ],
  "_source": false
}
```

But note that as we speak, this lookup field is not searchable:

```json
GET bytes-discuss-04/_search
{
  "runtime_mappings": {
    "category_description": {
        "type": "lookup", 
        "target_index": "bytes-discuss-categories", 
        "input_field": "category_name", 
        "target_field": "name", 
        "fetch_fields": [ "description" ]
    }
  },
  "fields": [
    "title",
    "category_name",
    "category_description"
  ],
  "query": {
    "match": {
      "category_description": "Restful"
    }
  }, 
  "_source": false
}
```

The solution is also to add this lookup field into the mapping and reindex the whole dataset to make it searchable.

## Conclusion

The runtime field could help you to solve some use cases that you did not think about when you first index your dataset. Remember that this will be slow and the more documents you have, the slower it will be.
When you are happy with an implementation, you can save it in the mapping and reindex the data to get back the performances you would expect from Elasticsearch.

Tomorrow my colleague Carly will speak about synonyms and the brand new 8.10 API we have to manage them efficiently. Subscribe to our channel. Bye.
