# Usage on aarch64 RaspberryPi 4 
used a RaspiBlitz v1.7 base image (based on https://www.raspberrypi.org/forums/viewtopic.php?p=1668160)

## Install Docker
```
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
echo \
  "deb [arch=arm64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get install docker-ce docker-ce-cli containerd.io

# add the default user to the docker group
sudo usermod -aG docker admin

# check (after relogin works without sudo)
sudo docker run hello-world
# Hello from Docker!
```

## Install docker-compose
```
# sudo curl -L "https://github.com/docker/compose/releases/download/1.29.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# sudo chmod +x /usr/local/bin/docker-compose

sudo pip3 install docker-compose
# https://docs.docker.com/compose/completion/
sudo curl \
    -L https://raw.githubusercontent.com/docker/compose/1.29.0/contrib/completion/bash/docker-compose \
    -o /etc/bash_completion.d/docker-compose

# check
docker-compose --version
# docker-compose version 1.29.0, build unknown
```

## Symlink working directory to the SSD
```
sudo systemctl stop docker
sudo systemctl stop docker.socket
sudo mv /var/lib/docker /mnt/hdd/
sudo ln -s  /mnt/hdd/docker /var/lib/docker
sudo systemctl start docker
sudo systemctl start docker.socket
```

## Download the repo and change to the aarch64 bitcoind
```
git clone https://github.com/bottlepay/lightning-benchmark
cd lightning-benchmark || exit 1

sed -i "s#kylemanna/bitcoind#openoms/aarch64-docker-bitcoind#" docker-compose-bbolt.yml
sed -i "s#kylemanna/bitcoind#openoms/aarch64-docker-bitcoind#" docker-compose-clightning.yml
sed -i "s#kylemanna/bitcoind#openoms/aarch64-docker-bitcoind#" docker-compose-eclair.yml
sed -i "s#kylemanna/bitcoind#openoms/aarch64-docker-bitcoind#" docker-compose-etcd-cluster.yml
sed -i "s#kylemanna/bitcoind#openoms/aarch64-docker-bitcoind#" docker-compose-etcd.yml
```
## Switch off the running applications
```
sudo systemctl stop bitcoind
sudo systemctl disable bitcoind
sudo systemctl stop lnd
sudo systemctl stop electrs
```

## Run
```
sudo chmod +x run.sh
run.sh lnd-bbolt | lnd-bbolt-keysend | lnd-etcd | lnd-etcd-cluster | clightning | eclair
```

## Results:
* RaspberryPi4 4GB RAM (quad core ARMv8)
* 1TB SSD connected to USB 3 with X825 extension board, ext4 filesystem
* RaspiBlitz 64bit base image (Raspberry OS - Debian Buster)

| Configuration | Transactions / sec | Avg latency (sec) |
|--|--|--|
|`clightning`|-| -  |
|`lnd-bbolt-keysend`| - | - |
|`lnd-bbolt`| 40.2603 | 2.45042 |
|`eclair`| - | - |
|`lnd-etcd`| - | - |
|`lnd-etcd-cluster`| - | - |

---
## Notes
### Manual build of the aarch64 docker-bitcoind
```
git clone https://github.com/kylemanna/docker-bitcoind.git
cd docker-bitcoind || exit 1
# add non sudo user
sudo adduser bitcoind
docker build --build-arg USER_ID=$( id -u bitcoind ) --build-arg GROUP_ID=$(  id -g bitcoind ) --build-arg ARCH=aarch64 .
```

### Use the local docker image with docker compose
* tag the locally built image
    ```
    docker tag <locally_built_image_ID> local_bitcoind
    ```
* list with `docker images`
  
* edit the `docker-compose-*.yml`  
 change `kylemanna/bitcoind` to `local_bitcoind`)
    ```
    services:
    bitcoind:
        image: local_bitcoind
        volumes:
        - ./bitcoin.conf:/bitcoin/.bitcoin/bitcoin.conf
    ```
### Upload the docker image
https://www.techrepublic.com/article/how-to-create-a-docker-image-and-push-it-to-docker-hub/
```
# running the benchmark will create the container
docker commit -m "add aarch64 bitcoind image" -a "openoms" lightning-benchmark_bitcoind_1 openoms/aarch64-docker-bitcoind
# sha256:2e3335bab7f94e10536f224f3d4343c2c42dcc5c635af52356aa3bc2decc5422
docker login
docker push openoms/aarch64-docker-bitcoind
# Using default tag: latest
# The push refers to repository [docker.io/openoms/aarch64-docker-bitcoind]
# ce8c73072aeb: Pushed 
# 51a645b37048: Pushed 
# b2307d1812fb: Pushed 
# ef8584c443a9: Pushed 
# a14e3cf239b6: Pushed 
# 37a64155d3cf: Pushed 
# 7137a365e1e4: Mounted from library/ubuntu 
# 9c64db1afb9e: Mounted from library/ubuntu 
# 8d4eb2300df0: Pushed 
# latest: digest: sha256:56c50e934c829dd6ed76f0b3e3019f0b5c6c04d4da7c75f45d010307f555242b size: 2195

```

### Calculate averages in bash
* paste the results to a `.txt`
* for TPS average:  
`awk '{ total += $7; count++ } END { print "tps average: "total/count }' tps.txt`
* for latency average:  
`awk '{ total += $11; count++ } END { print "latency average (sec): "total/count }' tps.txt`