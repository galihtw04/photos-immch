# photos-immch
> immich server digunakan untuk server penyimpanan foto seperti google photos

1. install docker on ubuntu
> ya immich ini akan kita install di dalam docker, karena yang disarankan menggunakan docker dan juga lebih simple untuk setupnya

- add repository
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

- install docker

```
apt install docker-ce docker-compose -y
```

2. install immich server

- create .env

```
mkdir immich-server && cd immich-server
```

```
cat <<"EOF"> .env
# You can find documentation for all the supported env variables at https://immich.app/docs/install/environment-variables

# The location where your uploaded files are stored
UPLOAD_LOCATION=/data-immich

# The Immich version to use. You can pin this to a specific version like "v1.71.0"
IMMICH_VERSION=release

# Connection secret for postgres. You should change it to a random password
DB_PASSWORD=postgres

# The values below this line do not need to be changed
###################################################################################
DB_HOSTNAME=immich_postgres
DB_USERNAME=postgres
DB_DATABASE_NAME=immich

REDIS_HOSTNAME=immich_redis
EOF
```

- create manifest yaml

```
cat <<"EOF"> docker-compose.yaml
version: '3.8'

#
# WARNING: Make sure to use the docker-compose.yml of the current release:
#
# https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
#
# The compose file on main may not be compatible with the latest release.
#

#name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    command: ['start.sh', 'immich']
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    ports:
      - 2283:3001
    depends_on:
      - redis
      - database
    restart: always

  immich-microservices:
    container_name: immich_microservices
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    # extends: # uncomment this section for hardware acceleration - see https://immich.app/docs/features/hardware-transcoding
    #   file: hwaccel.transcoding.yml
    #   service: cpu # set to one of [nvenc, quicksync, rkmpp, vaapi, vaapi-wsl] for accelerated transcoding
    command: ['start.sh', 'microservices']
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    depends_on:
      - redis
      - database
    restart: always

  immich-machine-learning:
    container_name: immich_machine_learning
    # For hardware acceleration, add one of -[armnn, cuda, openvino] to the image tag.
    # Example tag: ${IMMICH_VERSION:-release}-cuda
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
    # extends: # uncomment this section for hardware acceleration - see https://immich.app/docs/features/ml-hardware-acceleration
    #   file: hwaccel.ml.yml
    #   service: cpu # set to one of [armnn, cuda, openvino, openvino-wsl] for accelerated inference - use the `-wsl` version for WSL2 where applicable
    volumes:
      - model-cache:/cache
    env_file:
      - .env
    restart: always

  redis:
    container_name: immich_redis
    image: registry.hub.docker.com/library/redis:6.2-alpine@sha256:51d6c56749a4243096327e3fb964a48ed92254357108449cb6e23999c37773c5
    restart: always

  database:
    container_name: immich_postgres
    image: registry.hub.docker.com/tensorchord/pgvecto-rs:pg14-v0.2.0@sha256:90724186f0a3517cf6914295b5ab410db9ce23190a2d9d0b9dd6463e3fa298f0
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
    ports:
      - "9432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: always

  backup:
    container_name: immich_db_dumper
    image: prodrigestivill/postgres-backup-local
    env_file:
      - .env
    environment:
      POSTGRES_HOST: database
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      SCHEDULE: "0 0 1 */3 *"
      BACKUP_DIR: /db_dumps
    volumes:
      - ./db_dumps:/db_dumps
    depends_on:
      - database

  pgadmin:
    container_name: pgadmin_container1
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: galih@cloud.id
      PGADMIN_DEFAULT_PASSWORD: passowrd12345
    ports:
      - "5080:80"
    depends_on:
      - database
    restart: always

volumes:
  pgdata:
  model-cache:
EOF
```

- running docker compose

```
mkdir /data-immich # your store image photos
mkdir /db_dumps # store backup database
docker compose pull
docker compose up -d
docker compose ps -a
sleep 20
```

check container
```
docker compose ps -a
```

![image](https://github.com/galihtw04/photos-immch/assets/96242740/508c1715-6bef-4529-b5b1-a18c63b2c878)

- check curl access browser

![image](https://github.com/galihtw04/photos-immch/assets/96242740/8f463f34-e098-42fe-addc-fd77f37dd014)

> set email admin dan password

- Login

![image](https://github.com/galihtw04/photos-immch/assets/96242740/9839690a-2d3f-4e08-a336-a9304314943d)

![image](https://github.com/galihtw04/photos-immch/assets/96242740/01d9fc2c-c3de-426b-a11a-705ebf9fce35)

- testing upluod
![image](https://github.com/galihtw04/photos-immch/assets/96242740/a23523f1-0028-4928-9112-d52563495e6f)

3. Integration to cloudflare tunnel
> cloudflare digunakan untuk mem public server immich kita, agar bisa diakses dimana saja. Disini saya menggunakan cloudflare karena gratiss, kita hanya modal domain saja bisa mem public server private kita.

Untuk register domain ke cloudflare bisa mengikut langkah berikut

- register domain on cloudflare
![image](https://github.com/galihtw04/photos-immch/assets/96242740/49eedea5-0897-4602-8b06-f65a8266526e)

![image](https://github.com/galihtw04/photos-immch/assets/96242740/58bb60d6-42df-4506-ae5d-c9444cd2afb9)

![image](https://github.com/galihtw04/photos-immch/assets/96242740/a6b7e2e7-ad82-40d9-963d-28f6f09d7620)

![image](https://github.com/galihtw04/photos-immch/assets/96242740/cc3bef1f-083c-44b7-baaa-83ab56bdeb7c)

![image](https://github.com/galihtw04/photos-immch/assets/96242740/1a38fd7a-797a-4ff7-883d-e7b4f1ec98f1)

- change nameserver on domain mengarah ke cloudflare

![image](https://github.com/galihtw04/photos-immch/assets/96242740/bd6f1d28-d261-4aa0-a4c0-636524db7086)

![image](https://github.com/galihtw04/photos-immch/assets/96242740/e67993f6-feae-489e-9892-7540a4bf567d)

jika sudah teregister maka akan seperti berikut

![image](https://github.com/galihtw04/photos-immch/assets/96242740/aabbe449-e8ab-4a68-998d-cf104a3b82c5)

- create tunnel

subscribe zero trust free

![image](https://github.com/galihtw04/photos-immch/assets/96242740/d38a9a4f-4ff8-4df9-9d34-bf729a33599d)

![image](https://github.com/galihtw04/photos-immch/assets/96242740/9edc507c-6e1f-4301-b9ab-011af1227e87)

- create name tunnel(bebas)

![image](https://github.com/galihtw04/photos-immch/assets/96242740/e0d631a7-e2c9-4328-846d-9296592a491f)

- install tunnel on server private
> sesuaikan dengan os yang kalian gunakan

![image](https://github.com/galihtw04/photos-immch/assets/96242740/56b841ce-784e-477e-aa17-688361715a9f)

> copy command yang sudah di sediakan setelah kita memilih os sebelumnya.

![image](https://github.com/galihtw04/photos-immch/assets/96242740/1484df1d-5bb4-4729-9ec2-c0662e072807)


![image](https://github.com/galihtw04/photos-immch/assets/96242740/69d13ef6-150a-40ff-a5ed-792d137e3da0)


pastikan sudah seperti ini,

![image](https://github.com/galihtw04/photos-immch/assets/96242740/9a316e85-a5cb-4413-b127-64680aabb7ae)

- next buat configure Route Traffic

![image](https://github.com/galihtw04/photos-immch/assets/96242740/364b2da1-54c0-4f4f-87c2-6bfc92559871)

 - - Subdomain: pada parameter ini bebas bisa kalian isikan sesuai keinginan kalian
   - Type: ini bisa kalian sesuaikan dengan protocol server kalian, pada immich server menggunakan http
   - url: url yang digunakan adalah url untuk mengakses server immich, yang bisa diakses lewat server local

> Jika menurut kalian sudah sesuai maka klik save tunnel yang ada pada pojok bawah kanan.

![image](https://github.com/galihtw04/photos-immch/assets/96242740/2f55eea8-4e4d-4e13-abd6-455c497e1571)

- testing akses
untuk akses, kita bisa mengguanakan url subdomain yang telah kita buat tadi.

![image](https://github.com/galihtw04/photos-immch/assets/96242740/9e4aef03-f632-4267-9be4-816236a824cc)

# Done !!!

# Refrensi
- https://immich.app/docs/
