



### setup block storage

your nodes will need some extra space

```
docker plugin install rexray/dobs \
  DOBS_TOKEN=$DIGITALOCEAN_TOKEN \
  DOBS_REGION=$DIGITALOCEAN_REGION \
  LINUX_VOLUME_FILEMODE=0775
docker volume create --name=geth-$ETH1_NETWORK --opt=size=800 --driver=rexray/dobs
docker volume create --name=prysm-$BEACON_NETWORK --opt=size=50 --driver=rexray/dobs
```