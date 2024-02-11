#### Docker Stack 删除无用的服务

> 当某个服务从 yaml 文件中删除之后，更新 stack 时并不会删除无用的服务，需要加--prune 参数进行删除

```bash
sudo docker stack deploy -c /home/zjyq/zhxy/docker-compose.api.yml za --prune
```
