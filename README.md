# vault-splunk
Demo repo for vault-splunk integration
Credit: https://learn.hashicorp.com/tutorials/vault/monitor-telemetry-audit-splunk

* Clone the repo

```
$ git clone https://github.com/basingh/vault-splunk.git
```

* Under the folder initialize terraform 

```
$ terraform init
```

* Run terraform plan

```
$ terraform plan -out vault-metrics-lab.plan
```

* Apply terraform plan

```
$terraform apply vault-metrics-lab.plan
```

* Splunk image takes a while to come up, you want to make sure its on healthy state before proceeding. You can run following script to keep a watch on it


```
export splunk_ready=0
while [ $splunk_ready = 0 ]
  do
    if docker ps -f name=vtl-splunk --format "{{.Status}}" \
    | grep -q '(healthy)'
        then
            export splunk_ready=1
            echo "Splunk is ready."
        else
            printf "."
    fi
    sleep 5s
done
```

Or you can just use `docker ps`

```
$ docker ps -f name=vtl --format "table {{.Names}}\t{{.Status}}"
```

Following containers should come up here. The vault container will be in `unhealthy` state thats because its waiting to be initalized.

```
❯ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS                     PORTS                                                                           NAMES
ba491dbb281f   d3476693d7fd   "tini -- /bin/entryp…"   6 minutes ago   Up 6 minutes               5140/tcp, 24224/tcp                                                             vtl-fluentd
9a0a5fdb7c2c   e62c81adb6de   "docker-entrypoint.s…"   6 minutes ago   Up 6 minutes (unhealthy)   0.0.0.0:8200->8200/tcp                                                          vtl-vault
2fb8ba3941b1   3e8780e52e3f   "/sbin/entrypoint.sh…"   6 minutes ago   Up 6 minutes (healthy)     8065/tcp, 8088-8089/tcp, 8191/tcp, 9887/tcp, 0.0.0.0:8000->8000/tcp, 9997/tcp   vtl-splunk
7ecce17f029d   a2b6d302ca3c   "/entrypoint.sh tele…"   6 minutes ago   Up 6 minutes               8092/udp, 8125/udp, 8094/tcp                                                    vtl-telegraf
```

* Once all containers are up, time to initiate vault

```
## ssh into vault container
$ docker exec -it vtl-vault sh

## initialize vault
vault operator init \
    -key-shares=1 \
    -key-threshold=1 \
    | head -n3 \
    | cat > .vault-init

## unseal vault
vault operator unseal $(grep 'Unseal Key 1'  .vault-init | awk '{print $NF}')

## login with root token 
vault login -no-print \
$(grep 'Initial Root Token' .vault-init | awk '{print $NF}')

## token lookup
/ # vault token lookup
Key                 Value
---                 -----
accessor            WPhIh7WPljMhYKhteqSNrel1
creation_time       1614642294
creation_ttl        0s
display_name        root
entity_id           n/a
expire_time         <nil>
explicit_max_ttl    0s
id                  s.KF7yXC7Mud519NrXHD3DQxbU
meta                <nil>
num_uses            0
orphan              true
path                auth/token/root
policies            [root]
ttl                 0s
type                service
```

* Enable audit logging

```
vault audit enable file file_path=/vault/logs/vault-audit.log
```

* Following this you can access vault and splunk through UI on following local ports `http://127.0.0.1:8200/ui/` and `https://localhost:8000`

* You can login on splunk using credentials `username:admin password:vtl-password`

