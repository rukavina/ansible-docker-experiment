# ansible-docker-experiment

There are 3 machines:
* `lb` - load balancer `10.100.193.200`
* `app-01` - app server 1 `10.100.194.201`
* `app-02` - app server 2 `10.100.194.202`

```
        /- app-01 (apache+php/docker)
lb(nginx/docker)
        \_ app-02 (apache+php/docker)
```

`registrator` and `consul` runs on all nodes. registrator "watches" docker system and registers services info in `consul` db.
`consul-template` on `lb` node, generates proxy rules based on `consul` data and templates.

## provision nodes

```bash
vagrant up
ansible-playbook ansible/app.yml -i ansible/hosts/app
#haproxy version
ansible-playbook ansible/lb-haproxy.yml -i ansible/hosts/lb

#or nginx version
ansible-playbook ansible/lb-nginx.yml -i ansible/hosts/lb
```

## test service registry

```
vagrant ssh lb
curl 127.0.0.1:8500/v1/catalog/nodes
curl 127.0.0.1:8500/v1/catalog/services
curl 127.0.0.1:8500/v1/catalog/service/php1
curl 127.0.0.1:8500/v1/catalog/service/php2
```

## manual nginx template generation

`consul-template` roles runs bg process which watches `consul` and automatically updates config and reload proxy container.
So there's no need to generate files manually, but this just describes the process.

`vagrant ssh lb`

```bash
consul-template -template "/data/consul-template/nginx_locations_php_api.ctmpl:/tmp/locations_php_api.conf" -once
consul-template -template "/data/consul-template/nginx_upstreams_php_api.ctmpl:/tmp/upstreams_php_api.conf" -once

cp /tmp/locations_php_api.conf /data/nginx/includes/
cp /tmp/upstreams_php_api.conf /data/nginx/upstreams/

sudo docker kill -s HUP nginx
```

or 

```
bash /data/consul-template/regenerate-nginx-config.sh
sudo docker kill -s HUP nginx
```

## haproxy template generation

`consul-template` roles runs bg process which watches `consul` and automatically updates config and reload proxy container.
So there's no need to generate files manually, but this just describes the process.

`vagrant ssh lb`

```bash
#stop nginx container if running
sudo docker ps
sudo docker stop nginx

#generate config for running services
bash /data/consul-template/regenerate-haproxy-config.sh

#initial start
sudo docker restart haproxy

#while running
sudo docker kill -s HUP haproxy
```
## test calls

### from front-end:

For now this works only with nGinx!

in the host machine add to `/etc/hosts` the line:

```
127.0.0.1  api.horisen.test front1.horisen.test content.horisen.test
```

Now open in browser `http://front1.horisen.test:8081/` where you should see debug info with server used and headers.
Click on the button and watch the console, you should see api response json.

### from host machine:

```bash
curl http://127.0.0.1:8081/php1
curl http://127.0.0.1:8081/php2
```

###  from lb

```bash
curl http://127.0.0.1/php1
curl http://127.0.0.1/php2
```

### from app

to get ports run `curl 127.0.0.1:8500/v1/catalog/service/php1` and `curl 127.0.0.1:8500/v1/catalog/service/php2`

```bash
curl http://127.0.0.1:{port1}/php1
curl http://127.0.0.1:{port2}/php2
```

