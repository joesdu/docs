#### Docker Swarm 使用 NFS 进行数据共享

> 由于 docker swarm 是分布式集群,所以不能进行本地数据挂载.
> 当使用 docker volume 时,docker stack 能正常运行,但是服务数据只会存储在指定的 node 上,当服务迁移,数据会变为其他 node 上的数据.
> 简而言之,node 间数据是不共享的.

- 使用 NFS 来挂载存储卷

```yml
volumes:
  redis-data:
    driver: local
    driver_opts:
      type: nfs
      o: nfsvers=4,addr=192.168.2.112,rw
      device: ":/home/storage/mongo-data"
```
