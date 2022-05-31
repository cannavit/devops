# Create the Redis Cluster

[Redis](https://redis.io/) The open source, in-memory data store used by millions of developers as a database, cache, streaming engine, and message broker

![redis](https://www.bolchhanepal.com/wp-content/uploads/2019/02/Redis-labs-logo-1.png ':size=70%')

## Deploy Redis with Helm

[Redis(TM)](https://artifacthub.io/packages/helm/bitnami/redis) packaged by Bitnami
Redis(TM) is an open source, advanced key-value store. It is often referred to as a data structure server since keys can contain strings, hashes, lists, sets and sorted sets.

Disclaimer: Redis is a registered trademark of Redis Labs Ltd. Any rights therein are reserved to Redis Labs Ltd. Any use by Bitnami is for referential purposes only and does not indicate any sponsorship, endorsement, or affiliation between Redis Labs Ltd.


Add repository

    helm repo add bitnami https://charts.bitnami.com/bitnami

Install chart

    helm inspect values bitnami/redis > config_redis.yaml

Inside of the file change the password of the REDIS

    helm install -f  config_redis.yaml monitoring bitnami/redis --namespace example-production 

