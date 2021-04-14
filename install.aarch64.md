# Usage on aarch64 RaspberryPi 4 
used a RaspiBlitz v1.7 base image (based on https://www.raspberrypi.org/forums/viewtopic.php?p=1668160)

## Install Docker
```
# dependencies
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# add the docker repo
echo \
  "deb [arch=arm64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# install docker
sudo apt-get install docker-ce docker-ce-cli containerd.io

# add the default user to the docker group
sudo usermod -aG docker admin

# check (after relogin works without sudo)
sudo docker run hello-world
# Hello from Docker!
```

## Install docker-compose
```
sudo pip3 install docker-compose

# add bash completion  https://docs.docker.com/compose/completion/
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

## Download the repo and change to the aarch64 images
```
git clone https://github.com/bottlepay/lightning-benchmark
cd lightning-benchmark || exit 1

# bitcoind
sed -i "s#kylemanna/bitcoind#openoms/aarch64-docker-bitcoind#" docker-compose-bbolt.yml
sed -i "s#kylemanna/bitcoind#openoms/aarch64-docker-bitcoind#" docker-compose-clightning.yml
sed -i "s#kylemanna/bitcoind#openoms/aarch64-docker-bitcoind#" docker-compose-eclair.yml
sed -i "s#kylemanna/bitcoind#openoms/aarch64-docker-bitcoind#" docker-compose-etcd-cluster.yml
sed -i "s#kylemanna/bitcoind#openoms/aarch64-docker-bitcoind#" docker-compose-etcd.yml

# clightning
sed -i "s#elementsproject/lightningd:v0.9.3#openoms/clightning-linuxarm64v8:ade10e7fc4dacbb9d635b05152c7dc38c0896ce7#g" docker-compose-clightning.yml
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

Test results after 10,000 payments on the following machine:

* RaspberryPi4 4GB RAM (quad core ARMv8)
* 1TB Sandisk Ultra SSD 
* connected to USB 3.1 with an X825 extension board
* ext4 filesystem
* extra power for the SSD (dual 5V USB-C + barrel connector)
* passive cooling (50-55 degrees Celsius)
* RaspiBlitz 64bit base image (Raspberry OS - Debian Buster)
  

| Configuration | Transactions / sec | Avg latency (sec) |
|--|--|--|
|`clightning`| 34 | 2.5 |
|`lnd-bbolt-keysend`| 41 | 2.0 |
|`lnd-bbolt`| 40 | 2.5 |
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
# running a benchmark will create the container
# list containers
docker ps -a
# commit
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
### Build and upload the aarch64 clightning docker image
```
# build the docker image on the RPi4
git clone https://github.com/ElementsProject/lightning.git
cd lightning
docker build -t clightning-linuxarm64v8 -f contrib/linuxarm64v8.Dockerfile .

# run benchmark with the locally built image
sed -i "s#elementsproject/lightningd:v0.9.3#openoms/clightning-linuxarm64v8#g" docker-compose-clightning.yml
./run.sh clightning

# list containers
docker ps -a

# commit (tag with the latest commit has in https://github.com/ElementsProject/lightning)
docker commit -m "add aarch64 clightning image" -a "openoms" lightning-benchmark_clightning-alice_1 openoms/clightning-linuxarm64v8:ade10e7fc4dacbb9d635b05152c7dc38c0896ce7
# sha256:07f81a4e403fe330cbe2a96ed6393a17965e7bc3b3bdaebad56a8bca1f517084

# upload image to dockerhub 
docker login
docker push openoms/clightning-linuxarm64v8:ade10e7fc4dacbb9d635b05152c7dc38c0896ce7
# The push refers to repository [docker.io/openoms/clightning-linuxarm64v8]
# c8ea7e0b9521: Pushed 
# 8406f4e29e80: Pushed 
# 9b58aa07dc31: Pushed 
# 0202be3386c4: Pushed 
# 6081601076ab: Pushed 
# 723b4f743d0f: Pushed 
# 3c1c1ace6109: Pushed 
# 3ce8dc6f62f4: Pushed 
# f91f3ceae618: Pushed 
# cd6db239ed26: Mounted from library/debian 
# ade10e7fc4dacbb9d635b05152c7dc38c0896ce7: digest: sha256:ce36fa1a2b453485a9878bc2b654eff66aa098acd0bc748abbf5e9941a51a827 size: 2415
```

### Calculate the averages in bash
* paste the results to a `.txt`
* for TPS average:  
`awk '{ total += $7; count++ } END { print "tps average: "total/count }' tps.txt`
* for latency average:  
`awk '{ total += $11; count++ } END { print "latency average (sec): "total/count }' tps.txt`