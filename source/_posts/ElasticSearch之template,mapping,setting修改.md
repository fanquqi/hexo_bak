---
title: ElasticSearch使用
date: 2017-06-05 15:18:39
tags: ElasticSearch
categories: 运维工具
---

# ElasticSearch之template,mapping,setting修改

> 自搭建ELK以来一直跟elasticsearch打交道，但是只停留在基本会用，其他并没有更深入研究，其中发现问题越来越多，从之前字段自动analyzed ，导致画图是字段不能用，到后来分片问题。所以是时候study，mark一下。

## 基本命令
ES提供了丰富的API给我们使用通常我们可以使用curl来在线操作。也可以在kopf插件中操作（操作要谨慎）
例如： 
```
curl -XGET 'http://elasticsearch.example.com:9200/_all/_settings?pretty'
```

参考：http://spuder.github.io/2015/elasticsearch-commands/


## template 不重启更改

> 刚开始的时候，每次实验都去改/etc/elasticsearch/elasticsearch.yml配置文件。事实上在template里修改settings更方便而且灵活！当然最主要的，还是调节里面的properties设定，合理的控制store和analyze了。
logstash 模板

- 查看template
    + curl -XGET http://10.215.33.36:9200/_template

- 更改默认分片数量（[参考链接](http://spuder.github.io/2015/elasticsearch-default-shards/)）

> 索引分片数量是将一个索引一致性拆分成为n个部分，这样可以平均的存储到每个ES节点，方便更好的存储与查询。

### 查看默认template 
```
curl elasticsearch.example.com:9200/_template/logstash?pretty
```
结果显示如下
```
{
  "logstash" : {
    "order" : 0,
    "template" : "logstash-*",
    "settings" : {
      "index" : {
        "refresh_interval" : "5s"
      }
    },
    "mappings" : {
      "_default_" : {
        "dynamic_templates" : [ {
          "message_field" : {
            "mapping" : {
              "fielddata" : {
                "format" : "disabled"
              },
              "index" : "not_analyzed",
              "omit_norms" : true,
              "type" : "string"
            },
            "match_mapping_type" : "string",
            "match" : "message"
          }
        }, {
          "string_fields" : {
            "mapping" : {
              "fielddata" : {
                "format" : "disabled"
              },
              "index" : "not_analyzed",
              "omit_norms" : true,
              "type" : "string",
              "fields" : {
                "raw" : {
                  "ignore_above" : 256,
                  "index" : "not_analyzed",
                  "type" : "string"
                }
              }
            },
            "match_mapping_type" : "string",
            "match" : "*"
          }
        } ],
        "_all" : {
          "omit_norms" : true,
          "enabled" : true
        },
        "properties" : {
          "@timestamp" : {
            "type" : "date"
          },
          "geoip" : {
            "dynamic" : true,
            "properties" : {
              "ip" : {
                "type" : "ip"
              },
              "latitude" : {
                "type" : "float"
              },
              "location" : {
                "type" : "geo_point"
              },
              "longitude" : {
                "type" : "float"
              }
            }
          },
          "@version" : {
            "index" : "not_analyzed",
            "type" : "string"
          }
        }
      }
    },
    "aliases" : { }
  }
}
```

### 修改template，上传 
我们需要把这个json 重定向到一个文件中，其中没有用的部分去掉，加上需要改动的地方  
```
curl http://10.215.33.36:9200/_template/logstash?pretty > logstash_template.json
```
修改完的json如下`(最好把修改之前的配置备份下，这是运维的基本)`

```
{
    "template" : "logstash-*",
    "settings" : {
      "number_of_shards": 5
    },
    "mappings" : {
      "_default_" : {
        "dynamic_templates" : [ {
          "date_fields" : {
            "mapping" : {
              "format" : "dateOptionalTime",
              "doc_values" : true,
              "type" : "date"
            },
            "match" : "*",
            "match_mapping_type" : "date"
          }
        }, {
          "byte_fields" : {
            "mapping" : {
              "doc_values" : true,
              "type" : "byte"
            },
            "match" : "*",
            "match_mapping_type" : "byte"
          }
        }, {
          "double_fields" : {
            "mapping" : {
              "doc_values" : true,
              "type" : "double"
            },
            "match" : "*",
            "match_mapping_type" : "double"
          }
        }, {
          "float_fields" : {
            "mapping" : {
              "doc_values" : true,
              "type" : "float"
            },
            "match" : "*",
            "match_mapping_type" : "float"
          }
        }, {
          "integer_fields" : {
            "mapping" : {
              "doc_values" : true,
              "type" : "integer"
            },
            "match" : "*",
            "match_mapping_type" : "integer"
          }
        }, {
          "long_fields" : {
            "mapping" : {
              "doc_values" : true,
              "type" : "long"
            },
            "match" : "*",
            "match_mapping_type" : "long"
          }
        }, {
          "short_fields" : {
            "mapping" : {
              "doc_values" : true,
              "type" : "short"
            },
            "match" : "*",
            "match_mapping_type" : "short"
          }
        }, {
          "string_fields" : {
            "mapping" : {
              "index" : "not_analyzed",
              "omit_norms" : true,
              "doc_values" : true,
              "type" : "string"
            },
            "match" : "*",
            "match_mapping_type" : "string"
          }
        } ],
        "properties" : {
          "@version" : {
            "index" : "not_analyzed",
            "doc_values" : true,
            "type" : "string"
          }
        },
        "_all" : {
          "enabled" : true
        }
      }
    },
    "aliases" : { }

}
```
最后需要把这个配置push上去，方法如下`注意加"@"`
```
curl -XPUT http://10.215.33.36:9200/_template/logstash -d "@logstash_template_new.json"
```

第二天就可以看到5个分片`因为建好的index 配置是更改不了的 除非新建个新的把数据迁移下，我们没有这么着急，不如等到第二天`



## setting 不重启更改
- 配置副本数量
    - curl -XPUT http://10.215.33.36:9200/_settings -d '{"index":{"number_of_replicas":2}}'


## mapping 不重启更改
- 查看 nginx_access index的节点
    + 注意要在端口号后面加上索引的具体名称，支持正则，查看所有直接为空
    + curl -XGET '10.215.33.36:9200/logstash-nginx_access-*/_mapping?pretty'

- 更改属性
    + curl -XPOST '10.215.33.36:9200/logstash-nginx_access-*/_mapping' -d '{"mapping" :{"type" : "string", "index" : "not_analyzed"}}'



