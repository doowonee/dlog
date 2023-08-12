# Elasticsearch

JVM 힙 사이즈는 건들지 말라길래 그냥 기동함 어짜피 리소스 단독으로 es가 다 사용할태니 무방함.

```bash
# 이거 설정 안하면 서버 실행 준비 과정에서 죽음 sudo sysctl -p 하면 재부팅 안해도 됌
sudo vi /etc/sysctl.conf
vm.max_map_count=262144

# 데이터 노드 node.name IP 변경 주의
sudo docker run -d --name elasticsearch \
  --network host \
  --restart always \
  --log-driver json-file --log-opt max-size=1g \
  -e "cluster.name=cluster_name" \
  -e "cluster.initial_master_nodes=172.20.1.1,172.20.2.2,172.20.3.3" \
  -e "node.name=172.20.1.1" \
  -e "discovery.seed_hosts=172.20.1.1,172.20.2.2,172.20.3.3" \
  -e "bootstrap.memory_lock: true" \
  -e "network.host=_site_" \
  -e "node.ingest=false" \
  -v /home/ubuntu/data:/usr/share/elasticsearch/data \
  docker.elastic.co/elasticsearch/elasticsearch:7.16.3
sudo docker cp es_plugin.zip elasticsearch:/
sudo docker exec elasticsearch bin/elasticsearch-plugin install file:///es_plugin.zip
sudo docker restart elasticsearch
sudo docker logs -f elasticsearch

# 키바나 log 나올떄 까지 시간 좀 걸림 한 2분 정도
# ES랑 같은 버전 이어야함
# ELASTICSEARCH_HOSTS , 이후 띄어쓰기 하면 안됌
sudo docker stop kibana
sudo docker rm -f kibana
sudo docker run -d --name kibana \
  --network=host \
  --log-driver json-file --log-opt max-size=1g \
  -e 'ELASTICSEARCH_HOSTS=["http://172.20.1.1:9200","http://172.20.2.2:9200","http://172.20.3.3:9200"]' \
  docker.elastic.co/kibana/kibana:7.16.3
sudo docker logs -f kibana

# 코디네이터 노드 node.name IP 변경 주의
# 마스터는 데이터 노드만 지정 코디네이터 노드는 ingestion만 하게 설정
sudo docker stop elasticsearch
sudo docker rm -f elasticsearch
sudo docker run -d --name elasticsearch \
  --network host \
  --log-driver json-file --log-opt max-size=1g \
  -e "cluster.name=cluster_name" \
  -e "cluster.initial_master_nodes=172.20.1.1,172.20.2.2,172.20.3.3" \
  -e "node.name=172.20.5.5" \
  -e "discovery.seed_hosts=172.20.1.1,172.20.2.2,172.20.3.3" \
  -e "bootstrap.memory_lock: true" \
  -e "network.host=_site_" \
  -e "node.master=false" -e "node.data=false" \
  -v /home/ubuntu/data:/usr/share/elasticsearch/data \
  docker.elastic.co/elasticsearch/elasticsearch:7.16.3
sudo docker cp es_plugin.zip elasticsearch:/
sudo docker exec elasticsearch bin/elasticsearch-plugin install file:///es_plugin.zip
sudo docker restart elasticsearch
sudo docker logs -f elasticsearch
```

## 클러스터 사용시 검색 결과는 매번 달라진다

[문서](https://www.elastic.co/guide/en/elasticsearch/reference/master/consistent-scoring.html)를 보면 ES는 replica shard로 분산 검색을 할수있는데 이때 샤드마다 인덱싱 순서가 다를수 있으므로 score가 달라지고 그에 따라 검색 결과들의 순서가 달라질수 있다. 즉 페이징시 1번 페이지에 있던게 2번 페이지에도 있을수 있다는거다. 이걸 해결하려면 [preference](https://www.elastic.co/guide/en/elasticsearch/reference/master/search-search.html#search-preference)를 사용하면 된다.

## exit code with 137

`sudo dmesg | less` 입력후 `Out of memory` 검색 해보면 알수있듯이 메모리가 부족해서 OS가 컨테이너를 종료 시켜 버린다. 남는 메모리가 1.5 기가라 그런거 같은데 최소 4기가의 여유분은 되어야 하는것으로 보인다. 그래서 일단 [강제로 JVM 옵션](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/advanced-configuration.html#set-jvm-options)을 지정해서 최대 2기가만 사용하도록 지정하였다. 기존 ES data를 사제하지 않고 그대로 기동 해도 된다. EC2 인스턴스도 2기가 짜리인 t3.small 에서 4기가 짜리인 t3.medium으로 변경 하였다.

## mapping 팁

`enabled` 옵션은 인덱싱은 제외 하되 검색 결과에 `_source`에는 데이터가 존재 하도록 하는 옵션이라 인덱싱 성능을 올리고 인덱싱 대상 데이터를 줄여 최적화 할수 있지만 해당 필드에 대해서 `exists` 쿼리를 사용할수 없는 단점이 존재 한다.

`date` 필드는 기본이 `strict_date_optional_time`를 사용하는데 `yyyy-MM-ddTHH:mm:ss.SSSZ`, `yyyy-MM-dd` 형식을 지원 한다.

## analyzer

[analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/master/analyzer-anatomy.html) 구성요소

- [Character filter](https://www.elastic.co/guide/en/elasticsearch/reference/master/analysis-charfilters.html)
- [Tokenizer](https://www.elastic.co/guide/en/elasticsearch/reference/master/analysis-tokenizers.html)
- [Token filter](https://www.elastic.co/guide/en/elasticsearch/reference/master/analysis-tokenfilters.html)

<https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html#cjk-analyzer>를 보면 기본적으로 제공해주는 tokenizer와 token filter를 조합해서 언어 별로 적절한 analyzer 구성 방법이 서술되어있다.

[tokenizer](https://www.elastic.co/guide/en/elasticsearch/reference/master/analysis-tokenizers.html)와 [token filter](https://www.elastic.co/guide/en/elasticsearch/reference/master/analysis-tokenfilters.html)의 차이점은 전자는 문자열 스트림을 받아 token화 하는것인데 보통 단어 단위로 하게 된다. 그리고 필터는 이 `token`을 받아 수정 하거나 가공 하는 후 처리 작업인데 n-gram을 예로 min과 max를 2로 설정한 `tokenizer`만 사용하고 filter는 아무 옵션을 주지 않는다면 한글자만 검색자체가 안된다. 왜냐면 최소 2글자씩 tokenizing 되는데 1글자 이기 때문에 아예 무시가 된다는것이다. 하지만 `tokenzier`는 standard로 사용하고 filter를 통해 n-gram을 적용해주면 1글자 토큰도 사용할수 있게 된다.

`edge_ngram`는 문자열 처음부터 단순 최대 몇글자 까지 매칭 되는것이라 Ngram보다는 빠르다 또한 이때 띄어쓰기가 포함되면 단어로 나누어 지기떄문에 한국어나 영어에는 적합하지만 일본어에는 부적합 하다. 일본어 14글자가 띄어 쓰기가 없을 경우 `max_gram`: 10 이라면 뒤에 4글자는 검색이 되지 않는다. 때문에 <https://www.elastic.co/kr/blog/how-to-implement-japanese-full-text-search-in-elasticsearch>를 보다시피 일본어는 무조건 `Ngram` tokenizer를 사용해야 한다. 단 한국어일때도 띄어쓰기가 잘못 되어있다면 `ndge_ngram`으로 검색이 안되는 경우가 많을수있다. 예를들어 검색어는 `가수`인데 인덱싱된 단어는 `고전가수`일 경우이는 검색 결과에 나오지 않는다.

[stemmer](https://www.elastic.co/guide/en/elasticsearch/reference/master/analysis-stemmer-tokenfilter.html)는 알파벳 기반언어 특성을 이용해서 알고리즘으로 토큰을 어근, 어간등으로 분리하는 필터이다. [keyword_marker](https://www.elastic.co/guide/en/elasticsearch/reference/master/analysis-keyword-marker-tokenfilter.html)는 분리 대상에서 제외 하는 목록을 관리하는 필터이다.

기존꺼 보니까 mapping 할때 선언 하면 검색 할때 tokenizing을 알아서 하는줄 알았더니 그게 아니었어 인덱싱은 custom analyzer로 하지만 검색할때는 적용한 커스텀 아날라이저를 사용하지 않고 standard analyzer 를 사용하게 된다 그래서 아래 처럼 검색 쿼리에 명시적으로 analyzer를 지정 해주면 된다.

```json
GET /index_name/_search
{
  "explain": true, 
  "query": {
    "match": {
      "title": {
        "query": "search word",
        "analyzer": "custom_analyzer"
      }
    }
  }
}
```

## remote reindexing

리인덱싱으로 데이터를 받고자 하는 서버에 `config/elasticsearch.yaml` 파일에 `reindex.remote.whitelist: "172.30.9.9:9200"` 설정을 추가 한후 재기동 하고 아래의 쿼리를 실행 하면 된다. 로컬은 remote 없이 인덱스 이름만 사용 하면 된다.

```json
POST _reindex?wait_for_completion=false
{
  "source": {
    "remote": {
      "host": "http://172.30.9.9:9200"
    },
    "index": "some_index",
  },
  "dest": {
    "index": "some_index"
  }
}
```

실행하면 기다리지 않고 바로 task 정보가 반환 되는데 `GET _tasks/X9QMQfjYT8KK01OSOFM6NA:45`로 리인덱싱 얼만큼 해야하고 얼만큼 했는지 시간이랑 그 결과 계속 조회 가능함

## elasticdump

elastic search 데이터 export 유틸 사용법

```bash
# npm install -g elasticdump
#  raw data export 방법으로 json each row 형태로 뽑아 지게 된다
# 근데 왜 range인데 gte 아닌 애들도 탐색이 되지? 흠 숫자로 취급을 안하고 문자열로 취급해서 그런걸 수도 있다는데
elasticdump \
  --input=http://172.20.1.1:9200/index_name \
  --output="search_log.json_row" \
  --limit=10000 \
  --searchBody="{  \"_source\": [ \"title\",\"date\" ],  \"query\": {      \"range\": {      \"date\": {\"gte\": 16606060, \"lt\": 16706060   }    }  }}"

# elastic-dump로 export한 json each row를 csv로 변환 하는법
jq -s '.' search_log.json_row > out.json
jq --raw-output '.[] | ._source.meta | [.date, .title] | @csv' out.json > out.csv
```

## 버전 업그레이드

<https://www.elastic.co/guide/en/elasticsearch/reference/current/rolling-upgrades.html>를 보면 클러스터 무중단으로 업그레이드가 가능한데 메이저 버전이 같으면 데이터 마이그레이션 없이 컨테이너만 새로 기동 해서 업그레이드 가능하다. 단 주의 할점은 업그레이드 후 `cluster.routing.allocation.enable` 설정 원복 안하면 키바나 기동 안되고 데이터 노드 재기동 해도 인덱스들 리밸런싱도 안된다. 또한 ES plugin도 새로운 버전에 맞게 다시 패키징 해야 한다.

## 쿼리 노트

```javascript
// tasks 출력
GET _tasks

// 인덱스 목록
GET _cat/indices?v

// alias 목록
GET _cat/aliases?v

// alias 지정
POST /_aliases
{
  "actions" : [
    { "add" : { "index" : "i1", "alias" : "i" } },
    { "remove" : { "index" : "i1", "alias" : "i" } },
    { "add" : { "index" : "i2", "alias" : "i" } }
  ]
}

// 검색 결과 score외에 조건으로 수동 정렬
GET index_name/_search
{
  "sort": [
    {
      "broad_dt": {
        "order": "desc"
      }
    },
    {
      "_score": {
        "order": "desc"
      }
    }
  ], 
  "size": "15",
  "from": 0,
  "query": {
      "function_score": {
        "query": {
          "bool": {
            "filter": [
              {
                "match": {
                  "is_active": true
                }
              },
              {
                "match": {
                  "is_show": true
                }
              }
            ],
            "should": [
              {
                "term": {
                  "title": {
                    "value": "bts"
                  }
                }
              }
            ],
            "minimum_should_match": 1
          }
        },
        "functions": [
          {
            "gauss": {
              "broad_dt": {
                "scale": "90d",
                "offset": "15d",
                "decay": 0.1
              }
            }
          }
        ],
        "score_mode": "multiply",
        "boost_mode": "multiply"
      }
    }
}

// 데이터 수정
POST intex_name/_update/doc_id
{
  "doc": {
    "meta": {
      "flag.isuse": false,
      "parer" : {
        "show" : false
      }
    }
  }
}

// 랭킹 추출
GET /index_name/_search
{
  "query": {
      "range": {
      "broad_at": {
          "gte": 100,
          "lte": 500
      }
    }
  },
  "aggs": {
    "ranking": {
        "terms": {
            "size": 50,
            "field": "title"
        }
    }
  }
}

// 쿼리로 데이터 삽입
POST index_name/_doc/22293923
{
  "sid": {
    "id": "22293923"
  },
  "d": {
    "code": {
      "type": "somt",
      "content_type": 1
    },
    "title": "some_title"
  }
}
```
