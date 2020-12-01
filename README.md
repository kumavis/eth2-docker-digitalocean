
this is a configuration and instructions for setting up an eth2 validator on digital ocean

it is:
- incomplete
- insecure
- and provided without warranty or support

have fun

### what you need

##### software dependencies for your local machine

- linux
- docker
- docker-machine
- jq

##### dns name for your dashboard

for https. subdomains are fine. ssl certs are automated.

##### digital ocean account

and an api key

##### setup config + secrets

- copy `.env.example` to `.env`
- review configuration in `.env`, change as desired
- populate `.env` with:
  - `DO_TOKEN` (digital ocean api key)
  - `WEB_ADMIN_EMAIL` (your email)
  - `WEB_DASHBOARD_DOMAIN` (the domain for your monitoring dashboard)

### setting up the machine

##### create machine

create via docker-machine
```
(source .env;
  docker-machine create --driver digitalocean \
      --digitalocean-access-token $DO_TOKEN \
      --digitalocean-image $DO_BASE_IMAGE \
      --digitalocean-ipv6 \
      --digitalocean-monitoring \
      --digitalocean-size $DO_MACHINE_TYPE \
      --digitalocean-region $DO_REGION \
      $MACHINE_NAME
)
```

tell docker to use this machine
```
(source .env;
  eval $(docker-machine env $MACHINE_NAME)
)
```

install updates + restart
```
(source .env;
  docker-machine ssh $MACHINE_NAME 'apt update && apt upgrade -y && shutdown -r'
)
```

##### check machine security

verify ssh authentication rules
```
(source .env;
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


##### setup block storage

your nodes will need some extra space

create a volume on digital ocean https://cloud.digitalocean.com/volumes
and set it to attach to your machine and automatically format and mount

create volume
```
(source .env;
  curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer $DO_TOKEN" \
    -d "{\"size_gigabytes\":250, \"name\": \"$DO_VOLUME_NAME\", \"region\": \"$DO_REGION\", \"filesystem_type\": \"ext4\"}" \
    "https://api.digitalocean.com/v2/volumes"
)
```

attach volume
```
(source .env;
  DROPLETS_INFO=$(
    curl -X GET -H "Content-Type: application/json" -H "Authorization: Bearer $DO_TOKEN" \
    "https://api.digitalocean.com/v2/droplets"
  )
  DROPLET_ID=$(
    echo $DROPLETS_INFO | jq ".droplets[] | select(.name == \"$MACHINE_NAME\") | .id"
  )
  curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer $DO_TOKEN" \
    -d "{\"type\":\"attach\", \"volume_name\": \"$DO_VOLUME_NAME\", \"region\": \"$DO_REGION\", \"droplet_id\": \"$DROPLET_ID\"}" \
    "https://api.digitalocean.com/v2/volumes/actions"
)
```

verify: list volumes
```
(source .env;
  curl -X GET -H "Content-Type: application/json" -H "Authorization: Bearer $DO_TOKEN" \
    "https://api.digitalocean.com/v2/volumes?region=$DO_REGION" \
    | jq
)
```

remount volume to deterministic path
```
(source .env;
  docker-machine ssh $MACHINE_NAME "umount /dev/sda && mkdir /mnt/extra-data && mount -t ext4 /dev/sda /mnt/extra-data"
)
```

##### copy system config

this will copy files from your local machine to the remote machine

```
(source .env;
  docker-machine scp -d -r ./config $MACHINE_NAME:/var/
)
```

##### setup dns

set dns to point at your droplet.
you may alternatively setup a floating ip.

get the machine's ip address
```
(source .env;
  cat ~/.docker/machine/machines/$MACHINE_NAME/config.json | jq -r .Driver.IPAddress
)
```

### running it

##### boot system

start the system and show logs. close logs at any time.
```
docker-compose up -d && docker-compose logs
```

visit your url and set the grafana password (default: admin admin)


### validating

once geth is ready, you can start validating

##### import keys

copy keystore files into `./launchpad`, then to the remote machine
```
(source .env;
  docker-machine scp -d -r ./launchpad $MACHINE_NAME:/var
)
```

####### import accounts
```
docker-compose -f ./create-account.yaml run validator-import-launchpad
```

####### list accounts
```
docker-compose -f ./create-account.yaml run validator-list-accounts
```

##### helpful commands


##### syncing geth

syncing geth takes a long time, usually days. heres some utilities to measure it:

get syncing stats
```
docker-compose exec eth1 geth attach --datadir /data/geth --exec 'eth.syncing'
```

get geth data size
```
docker-compose exec eth1 du -sh /data/geth
```

##### helpful commands

get ip-address on host machine of container by name
```
docker inspect  -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker-compose ps -q node-exporter)
```