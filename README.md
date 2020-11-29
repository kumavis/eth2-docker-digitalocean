
### software dependencies for your local machine

- linux
- docker
- docker-machine
- jq

### setup config + secrets

- review `./env-config`
- create `./env-secret` from `./env-secret.example`
- populate `./env-secret` with `DO_TOKEN`

### create machine

create via docker-machine
```
(source ./env-secret; source ./env-config;
docker-machine create --driver digitalocean \
  --digitalocean-access-token $DO_TOKEN \
  --digitalocean-image "ubuntu-20-04-x64" \
  --digitalocean-ipv6 \
  --digitalocean-monitoring \
  --digitalocean-size "s-8vcpu-16gb" \
  --digitalocean-region $DO_REGION \
  $MACHINE_NAME
)
```

tell docker to use this machine
```
eval $(source ./env-config; docker-machine env $MACHINE_NAME)
```

**add DROPLET_ID to secrets**

```
(source ./env-secret; source ./env-config;
DROPLETS_INFO=$(
  curl -X GET -H "Content-Type: application/json" -H "Authorization: Bearer $DO_TOKEN" \
  "https://api.digitalocean.com/v2/droplets"
)
DROPLET_ID=$(
  echo $DROPLETS_INFO | jq ".droplets[] | select(.name == \"$MACHINE_NAME\") | .id"
)
echo "DO_DROPLET_ID=$DROPLET_ID" >> ./env-secret
)
```

install updates
```
(source ./env-config;
docker-machine ssh $MACHINE_NAME 'apt update && apt upgrade -y'
)
```

### check machine security

verify ssh authentication rules
```
(source ./env-config;
docker-machine ssh $MACHINE_NAME "sshd -T | grep authentication"
)
```

you should see. verify the yes/no match.
`authenticationmethods any` means authentication succeeds with one of any of the approved authentication methods (which is only `pubkeyauthentication`)
```
hostbasedauthentication no
pubkeyauthentication yes
kerberosauthentication no
gssapiauthentication no
passwordauthentication no
kbdinteractiveauthentication no
challengeresponseauthentication no
authenticationmethods any
```


### setup block storage

your nodes will need some extra space

create a volume on digital ocean https://cloud.digitalocean.com/volumes
and set it to attach to your machine and automatically format and mount

create volume
```
(source ./env-secret; source ./env-config;
curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer $DO_TOKEN" \
  -d "{\"size_gigabytes\":250, \"name\": \"$DO_VOLUME_NAME\", \"region\": \"$DO_REGION\", \"filesystem_type\": \"ext4\"}" \
  "https://api.digitalocean.com/v2/volumes"
)
```

attach volume
```
(source ./env-secret; source ./env-config;
curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer $DO_TOKEN" \
  -d "{\"type\":\"attach\", \"volume_name\": \"$DO_VOLUME_NAME\", \"region\": \"$DO_REGION\", \"droplet_id\": \"$DO_DROPLET_ID\"}" \
  "https://api.digitalocean.com/v2/volumes/actions"
)
```

verify: list volumes
```
(source ./env-secret; source ./env-config;
curl -X GET -H "Content-Type: application/json" -H "Authorization: Bearer $DO_TOKEN" \
  "https://api.digitalocean.com/v2/volumes?region=$DO_REGION" \
  | jq
)
```

remount volume to deterministic path
```
(source ./env-config;
docker-machine ssh $MACHINE_NAME "umount /dev/sda && mkdir /mnt/extra-data && mount -t ext4 /dev/sda /mnt/extra-data"
)
```

### copy system config

this will copy files from your local machine to the remote machine

```
(source ./env-config;
docker-machine scp -d -r ./config $MACHINE_NAME:/var/
)
```

### boot system


```
docker-compose up -d && docker-compose logs -f
```

get the machine's ip address
```
(source ./env-config;
cat ~/.docker/machine/machines/$MACHINE_NAME/config.json | jq -r .Driver.IPAddress
)
```

### import keys

copy keystore files into `./launchpad`, then to the remote machine
```
docker-machine scp -d -r ./launchpad $MACHINE_NAME:/var
```

##### import accounts
```
docker-compose -f ./create-account.yaml run validator-import-launchpad
```

##### list accounts
```
docker-compose -f ./create-account.yaml run validator-list-accounts
```

### syncing geth

syncing geth takes a long time, usually days. heres some utilities to measure it:

get syncing stats
```
docker-compose exec eth1 geth attach --datadir /data/geth --exec 'eth.syncing'
```

get geth data size
```
docker-compose exec eth1 du -sh /data/geth
```


### helpful commands

get ip-address on host machine of container by name
```
docker inspect  -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker-compose ps -q node-exporter)
```