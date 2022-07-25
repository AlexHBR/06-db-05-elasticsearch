# Домашнее задание к занятию "6.5. Elasticsearch"

## Задача 1

В этом задании вы потренируетесь в:
- установке elasticsearch
- первоначальном конфигурировании elastcisearch
- запуске elasticsearch в docker

Используя докер образ [elasticsearch:7](https://hub.docker.com/_/elasticsearch) как базовый:

- составьте Dockerfile-манифест для elasticsearch
- соберите docker-образ и сделайте `push` в ваш docker.io репозиторий
- запустите контейнер из получившегося образа и выполните запрос пути `/` c хост-машины

Требования к `elasticsearch.yml`:
- данные `path` должны сохраняться в `/var/lib` 
- имя ноды должно быть `netology_test`

В ответе приведите:
- текст Dockerfile манифеста
```commandline
FROM docker.elastic.co/elasticsearch/elasticsearch:7.17.5
LABEL MAINTAINER Zametaev.a

COPY --chown=elasticsearch:elasticsearch ./elasticsearch.yml /usr/share/elasticsearch/config/

RUN chown -R elasticsearch:elasticsearch /usr/share/elasticsearch/config/
RUN chown -R elasticsearch:elasticsearch /var/lib

RUN mkdir /usr/share/elasticsearch/snapshots
RUN chown -R elasticsearch:elasticsearch /usr/share/elasticsearch/snapshots

CMD ["sysctl -w vm.max_map_count=262144"]

CMD ["elasticsearch"]

EXPOSE 9200
```
```yaml
---
cluster.name: "docker-cluster"
discovery.type: single-node
network.host: 0.0.0.0
node.name: netology_test
path.data: /var/lib
path.repo: /usr/share/elasticsearch/snapshots
xpack.license.self_generated.type: basic
xpack.security.enabled: false
xpack.monitoring.collection.enabled: true
```


```commandline
docker build --tag=elastic_netology:7.17.15 .
docker tag d10a99638bf5 ahbrd2/elastic_netology:7.17.15
docker login
docker push ahbrd2/elastic_netology:7.17.15
docker run  -it -p 9200:9200  ahbrd2/elastic_netology:7.17.15
```
- ссылку на образ в репозитории dockerhub  
https://hub.docker.com/r/ahbrd2/elastic_netology

- ответ `elasticsearch` на запрос пути `/` в json виде
```commandline
root@serverd:/home/pp/05# curl -X GET "http://localhost:9200"
{
  "name" : "netology_test",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "SMrWnkaySTq20y6Jy6BzBQ",
  "version" : {
    "number" : "7.17.5",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "8d61b4f7ddf931f219e3745f295ed2bbc50c8e84",
    "build_date" : "2022-06-23T21:57:28.736740635Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

Подсказки:
- при сетевых проблемах внимательно изучите кластерные и сетевые настройки в elasticsearch.yml
- при некоторых проблемах вам поможет docker директива ulimit
- elasticsearch в логах обычно описывает проблему и пути ее решения
- обратите внимание на настройки безопасности такие как `xpack.security.enabled` 
- если докер образ не запускается и падает с ошибкой 137 в этом случае может помочь настройка `-e ES_HEAP_SIZE`
- при настройке `path` возможно потребуется настройка прав доступа на директорию

Далее мы будем работать с данным экземпляром elasticsearch.

## Задача 2

В этом задании вы научитесь:
- создавать и удалять индексы
- изучать состояние кластера
- обосновывать причину деградации доступности данных

Ознакомтесь с [документацией](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html) 
и добавьте в `elasticsearch` 3 индекса, в соответствии со таблицей:

| Имя | Количество реплик | Количество шард |
|-----|-------------------|-----------------|
| ind-1| 0 | 1 |
| ind-2 | 1 | 2 |
| ind-3 | 2 | 4 |

```bash
root@serverd:/home/pp/05# curl -X PUT "localhost:9200/ind-1?pretty" -H 'Content-Type: application/json
> ' -d'
> {
>   "settings": {
>     "index": {
>       "number_of_shards": 1,
>       "number_of_replicas": 0
>     }
>   }
> }
> '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-1"
}
root@serverd:/home/pp/05# curl -X PUT "localhost:9200/ind-2?pretty" -H 'Content-Type: application/json
> ' -d'
> {
>   "settings": {
>     "index": {
>       "number_of_shards": 2,
>       "number_of_replicas": 1
>     }
>   }
> }
> '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-2"
}
root@serverd:/home/pp/05# curl -X PUT "localhost:9200/ind-3?pretty" -H 'Content-Type: application/json
> ' -d'
> {
>   "settings": {
>     "index": {
>       "number_of_shards": 4,
>       "number_of_replicas": 2
>     }
>   }
> }
> '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-3"

```
Получите список индексов и их статусов, используя API и **приведите в ответе** на задание.
```
root@serverd:/home/pp# curl -X GET "localhost:9200/_cat/indices/ind-*"
green  open ind-1 Mm7wx4cLSJaFp5bta2lWYg 1 0 0 0 226b 226b
yellow open ind-3 cqxqMheKQzqA_l5rQIhqXw 4 2 0 0 904b 904b
yellow open ind-2 xvSgejnGQASERDuXf3IV2Q 2 1 0 0 452b 452b

root@serverd:/home/pp# curl -X GET "localhost:9200/_cat/shards/ind-*"
ind-2 1 p STARTED    0 226b 172.17.0.2 netology_test
ind-2 1 r UNASSIGNED
ind-2 0 p STARTED    0 226b 172.17.0.2 netology_test
ind-2 0 r UNASSIGNED
ind-1 0 p STARTED    0 226b 172.17.0.2 netology_test
ind-3 2 p STARTED    0 226b 172.17.0.2 netology_test
ind-3 2 r UNASSIGNED
ind-3 2 r UNASSIGNED
ind-3 1 p STARTED    0 226b 172.17.0.2 netology_test
ind-3 1 r UNASSIGNED
ind-3 1 r UNASSIGNED
ind-3 3 p STARTED    0 226b 172.17.0.2 netology_test
ind-3 3 r UNASSIGNED
ind-3 3 r UNASSIGNED
ind-3 0 p STARTED    0 226b 172.17.0.2 netology_test
ind-3 0 r UNASSIGNED
ind-3 0 r UNASSIGNED

```
Получите состояние кластера `elasticsearch`, используя API.
```shell
root@serverd:/home/pp# curl -X GET "http://localhost:9200/_cluster/health"
{"cluster_name":"docker-cluster","status":"yellow","timed_out":false,"number_of_nodes":1,"number_of_data_nodes":1,"active_primary_shards":11,"active_shards":11,"relocating_shards":0,"initializing_shards":0,"unassigned_shards":10,"delayed_unassigned_shards":0,"number_of_pending_tasks":0,"number_of_in_flight_fetch":0,"task_max_waiting_in_queue_millis":0,"active_shards_percent_as_number":52.38095238095239}
```

Как вы думаете, почему часть индексов и кластер находится в состоянии yellow?
>- все primary шарды в состоянии assigned. Часть secondary - шард в состоянии unassigned.
>- для 'ind-2' и 'ind-3' создано большее количество реплик и шард, чем есть на самом деле.  

Удалите все индексы.
```shell
root@serverd:/home/pp# curl -X DELETE "localhost:9200/ind-*"
{"acknowledged":true}
```

**Важно**

При проектировании кластера elasticsearch нужно корректно рассчитывать количество реплик и шард,
иначе возможна потеря данных индексов, вплоть до полной, при деградации системы.

## Задача 3

В данном задании вы научитесь:
- создавать бэкапы данных
- восстанавливать индексы из бэкапов

Создайте директорию `{путь до корневой директории с elasticsearch в образе}/snapshots`.

Используя API [зарегистрируйте](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-register-repository.html#snapshots-register-repository) 
данную директорию как `snapshot repository` c именем `netology_backup`.
>Инфраструктура подготовлена в Dockerfile

**Приведите в ответе** запрос API и результат вызова API для создания репозитория.
```shell
root@serverd:/home/pp# curl -X PUT "localhost:9200/_snapshot/netology_backup" -H 'Content-Type: application/json' -d'
> {
>   "type": "fs",
>   "settings": {
>     "location": "/usr/share/elasticsearch/snapshots"
>   }
> }
> '
{"acknowledged":true}
```
Создайте индекс `test` с 0 реплик и 1 шардом и **приведите в ответе** список индексов.
```shell
root@serverd:/home/pp# curl -X PUT "localhost:9200/test?pretty" -H 'Content-Type: application/json' -d'
> {
>   "settings": {
>     "index": {
>       "number_of_shards": 1,
>       "number_of_replicas": 0
>     }
>   }
> }
> '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "test"
}
```

[Создайте `snapshot`](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-take-snapshot.html) 
состояния кластера `elasticsearch`.
```shell
root@serverd:/home/pp# curl -X PUT "localhost:9200/_snapshot/netology_backup/%3Cmy_snapshot_%7Bnow%2Fd%7D%3E?pretty"
{
  "accepted" : true
}
```
**Приведите в ответе** список файлов в директории со `snapshot`ами.
```shell
root@59901fb7ee47:/usr/share/elasticsearch/snapshots# ls -l
total 48
-rw-rw-r-- 1 elasticsearch root  1702 Jul 24 21:47 index-0
-rw-rw-r-- 1 elasticsearch root     8 Jul 24 21:47 index.latest
drwxrwxr-x 7 elasticsearch root  4096 Jul 24 21:46 indices
-rw-rw-r-- 1 elasticsearch root 29299 Jul 24 21:46 meta-XQRF_nRdRmOncUjhmD-pgQ.dat
-rw-rw-r-- 1 elasticsearch root   789 Jul 24 21:46 snap-XQRF_nRdRmOncUjhmD-pgQ.dat
```
Удалите индекс `test` и создайте индекс `test-2`. **Приведите в ответе** список индексов.
```shell
root@serverd:/home/pp# curl -X DELETE "localhost:9200/test"
{"acknowledged":true}

root@serverd:/home/pp# curl -X PUT "localhost:9200/test-2?pretty" -H 'Content-Type: application/json' -d'
> {
>   "settings": {
>     "index": {
>       "number_of_shards": 1,
>       "number_of_replicas": 0
>     }
>   }
> }
> '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "test-2"
}

root@serverd:/home/pp#  curl -X GET "localhost:9200/_cat/indices/*"
green open test-2                      aG2TT-dASgGhj9xflQ836w 1 0    0    0   226b   226b
green open .geoip_databases            gGM1Q9AkQsK6h2PTgfAlLw 1 0   40    0 37.7mb 37.7mb
green open .monitoring-es-7-2022.07.24 ucDzmfu8SBik_BV_bvePhA 1 0 3653 4595  3.5mb  3.5mb

```
[Восстановите](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-restore-snapshot.html) состояние
кластера `elasticsearch` из `snapshot`, созданного ранее. 

**Приведите в ответе** запрос к API восстановления и итоговый список индексов.

```shell
 root@serverd:/home/pp# curl -X GET "localhost:9200/_snapshot/netology_backup/*?verbose=false&pretty"
{
  "snapshots" : [
    {
      "snapshot" : "my_snapshot_2022.07.24",
      "uuid" : "XQRF_nRdRmOncUjhmD-pgQ",
      "repository" : "netology_backup",
      "indices" : [
        ".ds-.logs-deprecation.elasticsearch-default-2022.07.24-000001",
        ".ds-ilm-history-5-2022.07.24-000001",
        ".geoip_databases",
        ".monitoring-es-7-2022.07.24",
        "test"
      ],
      "data_streams" : [ ],
      "state" : "SUCCESS"
    }
  ],
  "total" : 1,
  "remaining" : 0
}

 
 curl -X POST  "localhost:9200/.*/_close?pretty"
 
 
 root@serverd:/home/pp# curl -X POST "localhost:9200/_snapshot/netology_backup/my_snapshot_2022.07.24/_restore?pretty"
{
  "accepted" : true
}
root@serverd:/home/pp# curl -X GET "localhost:9200/_cat/indices/*"
green open .geoip_databases            3oWDPDsORNOVOAECJi0ZNQ 1 0   40    0 37.7mb 37.7mb
green open test-2                      aG2TT-dASgGhj9xflQ836w 1 0    0    0   226b   226b
green open test                        Tfy3MoiOQniQARj3MvrQ1A 1 0    0    0   226b   226b
green open .monitoring-es-7-2022.07.24 ucDzmfu8SBik_BV_bvePhA 1 0 3653 4595    2mb    2mb

root@serverd:/home/pp# curl -X GET "localhost:9200/_cluster/health?pretty"
{
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 6,
  "active_shards" : 6,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
root@serverd:/home/pp# curl -X GET "localhost:9200/test/_recovery?pretty"
{
  "test" : {
    "shards" : [
      {
        "id" : 0,
        "type" : "SNAPSHOT",
        "stage" : "DONE",
        "primary" : true,
        "start_time_in_millis" : 1658722234428,
        "stop_time_in_millis" : 1658722234582,
        "total_time_in_millis" : 154,
        "source" : {
          "repository" : "netology_backup",
          "snapshot" : "my_snapshot_2022.07.24",
          "version" : "7.17.5",
          "index" : "test",
          "restoreUUID" : "ynO_uOITTMGbMZYLdIws7A"
        },
        "target" : {
          "id" : "yMjVnTmfTVaPXPDhilKYeA",
          "host" : "172.17.0.2",
          "transport_address" : "172.17.0.2:9300",
          "ip" : "172.17.0.2",
          "name" : "netology_test"
        },
        "index" : {
          "size" : {
            "total_in_bytes" : 226,
            "reused_in_bytes" : 0,
            "recovered_in_bytes" : 226,
            "recovered_from_snapshot_in_bytes" : 0,
            "percent" : "100.0%"
          },
          "files" : {
            "total" : 1,
            "reused" : 0,
            "recovered" : 1,
            "percent" : "100.0%"
          },
          "total_time_in_millis" : 108,
          "source_throttle_time_in_millis" : 0,
          "target_throttle_time_in_millis" : 0
        },
        "translog" : {
          "recovered" : 0,
          "total" : 0,
          "percent" : "100.0%",
          "total_on_start" : 0,
          "total_time_in_millis" : 33
        },
        "verify_index" : {
          "check_index_time_in_millis" : 0,
          "total_time_in_millis" : 0
        }
      }
    ]
  }
}

 ```
Подсказки:
- возможно вам понадобится доработать `elasticsearch.yml` в части директивы `path.repo` и перезапустить `elasticsearch`

---

### Как cдавать задание

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
