



### setup block storage

your nodes will need some extra space

create a volume on digital ocean https://cloud.digitalocean.com/volumes
and set it to attach to your machine and automatically format and mount


### copy system config
```
docker-machine scp -d -r ./config $REMOTE_MACHINE:/var/
```

### import keys

copy keystore files into `./launchpad`, then to the remote machine
```
docker-machine scp -d -r ./launchpad $REMOTE_MACHINE:/var
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