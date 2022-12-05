# Домашнее задание к занятию "6.5. Elasticsearch" - Карпов Андрей, Devops-21

## Задача 1

В этом задании вы потренируетесь в:
- установке elasticsearch
- первоначальном конфигурировании elastcisearch
- запуске elasticsearch в docker

Используя докер образ [centos:7](https://hub.docker.com/_/centos) как базовый и 
[документацию по установке и запуску Elastcisearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/targz.html):

- составьте Dockerfile-манифест для elasticsearch
- соберите docker-образ и сделайте `push` в ваш docker.io репозиторий
- запустите контейнер из получившегося образа и выполните запрос пути `/` c хост-машины

Требования к `elasticsearch.yml`:
- данные `path` должны сохраняться в `/var/lib`
- имя ноды должно быть `netology_test`

В ответе приведите:
- текст Dockerfile манифеста
- ссылку на образ в репозитории dockerhub
- ответ `elasticsearch` на запрос пути `/` в json виде

Подсказки:
- возможно вам понадобится установка пакета perl-Digest-SHA для корректной работы пакета shasum
- при сетевых проблемах внимательно изучите кластерные и сетевые настройки в elasticsearch.yml
- при некоторых проблемах вам поможет docker директива ulimit
- elasticsearch в логах обычно описывает проблему и пути ее решения

Далее мы будем работать с данным экземпляром elasticsearch.

### Ответ:
Сначала я сделал свой докерфайл, но так и не смог победить ошибку доступа по ssl, видимо нужно было сгенерировать сертификаты и пр.
```dockerfile
From centos:7
RUN yum install -y java-11-openjdk java-11-openjdk-devel wget
ENV JAVA_HOME /etc/alternatives/jre
RUN rpm --import https://artifacts.opensearch.org/publickeys/opensearch.pgp && \ 
wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.4.0/opensearch-2.4.0-linux-x64.rpm \ 
&& yum install -y opensearch-2.4.0-linux-x64.rpm && rm opensearch-2.4.0-linux-x64.rpm && rm -rf /var/cache/yum
RUN chown -R 1000 /var/log/opensearch && chown -R 1000 /etc/opensearch && chown -R 1000 /var/lib/opensearch
EXPOSE 9200
CMD /usr/share/opensearch/bin/opensearch
```
Так что потом я подсмотрел как делает разработчик
```dockerfile
########################### Stage 0 ########################
FROM centos:7 AS linux_stage_0

ARG UID=1000
ARG GID=1000
ARG TEMP_DIR=/tmp/opensearch
ARG OPENSEARCH_HOME=/usr/share/opensearch
ARG OPENSEARCH_PATH_CONF=$OPENSEARCH_HOME/config
ARG SECURITY_PLUGIN_DIR=$OPENSEARCH_HOME/plugins/opensearch-security
ARG PERFORMANCE_ANALYZER_PLUGIN_CONFIG_DIR=$OPENSEARCH_PATH_CONF/opensearch-performance-analyzer

# Update packages
# Install the tools we need: tar and gzip to unpack the OpenSearch tarball, and shadow-utils to give us `groupadd` and `useradd`.
# Install which to allow running of securityadmin.sh
RUN yum update -y && yum install -y tar gzip shadow-utils which && yum clean all

# Create an opensearch user, group, and directory
RUN groupadd -g $GID opensearch && \
    adduser -u $UID -g $GID -d $OPENSEARCH_HOME opensearch && \
    mkdir $TEMP_DIR

# Prepare working directory
# Copy artifacts and configurations to corresponding directories
COPY * $TEMP_DIR/
RUN ls -l $TEMP_DIR && \
    tar -xzpf /tmp/opensearch/opensearch-`uname -p`.tgz -C $OPENSEARCH_HOME --strip-components=1 && \
    mkdir -p $OPENSEARCH_HOME/data && chown -Rv $UID:$GID $OPENSEARCH_HOME/data && \
    if [[ -d $SECURITY_PLUGIN_DIR ]] ; then chmod -v 750 $SECURITY_PLUGIN_DIR/tools/* ; fi && \
    if [[ -d $PERFORMANCE_ANALYZER_PLUGIN_CONFIG_DIR ]] ; then cp -v $TEMP_DIR/performance-analyzer.properties $PERFORMANCE_ANALYZER_PLUGIN_CONFIG_DIR; fi && \
    cp -v $TEMP_DIR/opensearch-docker-entrypoint.sh $TEMP_DIR/opensearch-onetime-setup.sh $OPENSEARCH_HOME/ && \
    cp -v $TEMP_DIR/log4j2.properties $TEMP_DIR/opensearch.yml $TEMP_DIR/*.pem $OPENSEARCH_PATH_CONF/ && \
    ls -l $OPENSEARCH_HOME && \
    rm -rf $TEMP_DIR


########################### Stage 1 ########################
# Copy working directory to the actual release docker images
FROM centos:7

ARG UID=1000
ARG GID=1000
ARG OPENSEARCH_HOME=/usr/share/opensearch

# Update packages
# Install the tools we need: tar and gzip to unpack the OpenSearch tarball, and shadow-utils to give us `groupadd` and `useradd`.
# Install which to allow running of securityadmin.sh
RUN yum update -y && yum install -y tar gzip shadow-utils which && yum clean all

# Create an opensearch user, group
RUN groupadd -g $GID opensearch && \
    adduser -u $UID -g $GID -d $OPENSEARCH_HOME opensearch

# Copy from Stage0
COPY --from=linux_stage_0 --chown=$UID:$GID $OPENSEARCH_HOME $OPENSEARCH_HOME
WORKDIR $OPENSEARCH_HOME

# Set $JAVA_HOME
RUN echo "export JAVA_HOME=$OPENSEARCH_HOME/jdk" >> /etc/profile.d/java_home.sh && \
    echo "export PATH=\$PATH:\$JAVA_HOME/bin" >> /etc/profile.d/java_home.sh

ENV JAVA_HOME=$OPENSEARCH_HOME/jdk
ENV PATH=$PATH:$JAVA_HOME/bin:$OPENSEARCH_HOME/bin

# Add k-NN lib directory to library loading path variable
ENV LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$OPENSEARCH_HOME/plugins/opensearch-knn/lib"

# Change user
USER $UID

# Setup OpenSearch
# Disable security demo installation during image build, and allow user to disable during startup of the container
# Enable security plugin during image build, and allow user to disable during startup of the container
ARG DISABLE_INSTALL_DEMO_CONFIG=true
ARG DISABLE_SECURITY_PLUGIN=false
RUN ./opensearch-onetime-setup.sh

# Expose ports for the opensearch service (9200 for HTTP and 9300 for internal transport) and performance analyzer (9600 for the agent and 9650 for the root cause analysis component)
EXPOSE 9200 9300 9600 9650

# CMD to run
ENTRYPOINT ["./opensearch-docker-entrypoint.sh"]
CMD ["opensearch"]
```

Ответ `opensearch`:
```bash
root@debian:/home/debian/elastic# curl -k -u admin:admin https://localhost:9200
{
  "name" : "netology_test",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "0YA4igFuQtWUZbvDLVe-zg",
  "version" : {
    "distribution" : "opensearch",
    "number" : "2.4.0",
    "build_type" : "tar",
    "build_hash" : "744ca260b892d119be8164f48d92b8810bd7801c",
    "build_date" : "2022-11-15T04:42:29.671309257Z",
    "build_snapshot" : false,
    "lucene_version" : "9.4.1",
    "minimum_wire_compatibility_version" : "7.10.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "The OpenSearch Project: https://opensearch.org/"
}
```
Ссылка на образ в репозитории dockerhub
<a href='https://hub.docker.com/r/askarpoff/centos-7-opensearch'>https://hub.docker.com/r/askarpoff/centos-7-opensearch</a>

# Задача 2

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

Получите список индексов и их статусов, используя API и **приведите в ответе** на задание.

Получите состояние кластера `elasticsearch`, используя API.

Как вы думаете, почему часть индексов и кластер находится в состоянии yellow?

Удалите все индексы.

**Важно**

При проектировании кластера elasticsearch нужно корректно рассчитывать количество реплик и шард,
иначе возможна потеря данных индексов, вплоть до полной, при деградации системы.

### Ответ:
3 индекса
```bash
root@debian:/home/debian# curl -k -u admin:admin -X PUT "https://localhost:9200/ind-1?pretty" -H 'Content-Type: application/json' -d'
> {
>   "settings": {
>     "number_of_shards": 1,
>     "number_of_replicas": 0
>   }
> }
> '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-1"
}
root@debian:/home/debian# curl -k -u admin:admin -X PUT "https://localhost:9200/ind-2?pretty" -H 'Content-Type: application/json' -d'
> {
>   "settings": {
>     "number_of_shards": 2,
>     "number_of_replicas": 1
>   }
> }
> '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-2"
}
root@debian:/home/debian# curl -k -u admin:admin -X PUT "https://localhost:9200/ind-3?pretty" -H 'Content-Type: application/json' -d'
> {
>   "settings": {
>     "number_of_shards": 4,
>     "number_of_replicas": 2
>   }
> }
> '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-3"
}
```
Список индексов и их статусов
```bash
root@debian:/home/debian# curl -k -u admin:admin -X GET "https://localhost:9200/_cat/indices?v"
health status index                        uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   security-auditlog-2022.12.04 lZTIkkfeQIaQ7Nd615G_7Q   1   1         18            0    160.3kb        160.3kb
green  open   ind-1                        S5w-jl1jSuGVc4Xl5PPVbw   1   0          0            0       208b           208b
green  open   .opendistro_security         01ppSzVsQrSa-YtIKmZyrw   1   0         10            0     71.7kb         71.7kb
yellow open   ind-3                        N-TddOwVS7uliqsi0hPKQw   4   2          0            0       832b           832b
yellow open   ind-2                        bwe6cBDLRFOYdZhfob08kA   2   1          0            0       416b           416b
```

Состояние кластера
```bash
root@debian:/home/debian# curl -k -u admin:admin -X GET "https://localhost:9200/_cluster/health?pretty"
{
  "cluster_name" : "opensearch-cluster",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "discovered_master" : true,
  "discovered_cluster_manager" : true,
  "active_primary_shards" : 9,
  "active_shards" : 9,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 11,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 45.0
}
```
Часть индексов и кластер находится в состоянии yellow: создали индексы с репликами,а нода толька одна, некуда реплицировать, соответсвенно "желтый" статус.

Удалите все индексы:
Просто так не прокатывает, потому что в opensearch.yml
```yaml
action.destructive_requires_name: true
```
```bash
root@debian:/home/debian# curl -k -u admin:admin -X DELETE "https://localhost:9200/_all?pretty"
{
  "error" : {
    "root_cause" : [
      {
        "type" : "security_exception",
        "reason" : "no permissions for [] and User [name=admin, backend_roles=[admin], requestedTenant=null]"
      }
    ],
    "type" : "security_exception",
    "reason" : "no permissions for [] and User [name=admin, backend_roles=[admin], requestedTenant=null]"
  },
  "status" : 403
}
```
Так что по-одному - типа  
```bash
curl -k -u admin:admin -X DELETE "https://localhost:9200/ind-1?pretty"
{
  "acknowledged" : true
}
```
## Задача 3

В данном задании вы научитесь:
- создавать бэкапы данных
- восстанавливать индексы из бэкапов

Создайте директорию `{путь до корневой директории с elasticsearch в образе}/snapshots`.

Используя API [зарегистрируйте](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-register-repository.html#snapshots-register-repository) 
данную директорию как `snapshot repository` c именем `netology_backup`.

**Приведите в ответе** запрос API и результат вызова API для создания репозитория.

Создайте индекс `test` с 0 реплик и 1 шардом и **приведите в ответе** список индексов.

[Создайте `snapshot`](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-take-snapshot.html) 
состояния кластера `elasticsearch`.

**Приведите в ответе** список файлов в директории со `snapshot`ами.

Удалите индекс `test` и создайте индекс `test-2`. **Приведите в ответе** список индексов.

[Восстановите](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-restore-snapshot.html) состояние
кластера `elasticsearch` из `snapshot`, созданного ранее. 

**Приведите в ответе** запрос к API восстановления и итоговый список индексов.

Подсказки:
- возможно вам понадобится доработать `elasticsearch.yml` в части директивы `path.repo` и перезапустить `elasticsearch`

### Ответ:
Запрос API и результат вызова API для создания репозитория.

Добавил - path.repo=/usr/share/opensearch/snapshots
```bash
root@debian:/home/debian# curl -k -u admin:admin -X PUT "https://localhost:9200/_snapshot/netology_backup?pretty" -H 'Content-Type: application/json' -d'
> {
>   "type": "fs",
>   "settings": {
>     "location": "/usr/share/opensearch/snapshots"
>   }
> }
> '
{
  "acknowledged" : true
}
```
Список индексов
```bash
root@debian:/home/debian# curl -k -u admin:admin -X GET "https://localhost:9200/_cat/indices?v"
health status index                        uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   security-auditlog-2022.12.05 z4x_X56MQnOJSvmLSPNclw   1   1          4            0     60.9kb         60.9kb
green  open   test                         RU321FhgQZKwqSXAVEvZ1Q   1   0          0            0       208b           208b
green  open   .opendistro_security         B2vobB6xR1CmGMKPE6UhNw   1   0         10            0     71.7kb         71.7kb
```
Создайте snapshot
```bash
root@debian:/home/debian# curl -k -u admin:admin -X PUT "https://localhost:9200/_snapshot/netology_backup/%3Csnapshot_%7Bnow%2Fd%7D%3E?pretty"
{
  "accepted" : true
}
```
Список файлов из директории со снапшотами:
```bash
[opensearch@737f8c6b30d4 snapshots]$ ll
total 20
-rw-rw-r-- 1 opensearch opensearch  952 Dec  5 08:38 index-0
-rw-rw-r-- 1 opensearch opensearch    8 Dec  5 08:38 index.latest
drwxrwxr-x 5 opensearch opensearch 4096 Dec  5 08:38 indices
-rw-rw-r-- 1 opensearch opensearch  371 Dec  5 08:38 meta-IG6aun0FSvWB8RFevetRwA.dat
-rw-rw-r-- 1 opensearch opensearch  326 Dec  5 08:38 snap-IG6aun0FSvWB8RFevetRwA.dat
```
Удалите индекс `test` и создайте индекс `test-2`
```

```
Запрос к API восстановления и итоговый список индексов
```

```
