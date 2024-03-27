# Backup & Restore
Disini saya akan mencoba untuk backup dan restore immich server kita, casenya adalah semisal data ada masalah, atau kita ingin migrasi/integrasi ke server lain.

## create backup

```
apt install zip
```

```
mkdir ~/immich-backup && cd ~/immich-backup
docker exec -t immich_postgres pg_dumpall -c -U postgres | gzip > "./dump.sql.gz" ## backup database
tar -cvf data-immich.tar /data-immich # backup file on immich(photos,video.dll)
```
![image](https://github.com/galihtw04/photos-immch/assets/96242740/db484e9b-05e6-44db-be7e-908a2589035d)

- restore immich server

  - remove/down docker compose
untuk menguji restore kita akan scale down container atau menghapus container

```
cd ~/immich-server
docker compose ps -a
docker compose down -v
```

![image](https://github.com/galihtw04/photos-immch/assets/96242740/5667a7f1-db9b-417b-b2d1-bc8774916c0b)

  - copy file .env dan docker-compose.yaml

```
cp docker-compose.yaml ~/immich-backup/ && cp .env ~/immich-backup/
cd ~/immich-backup/
```

![image](https://github.com/galihtw04/photos-immch/assets/96242740/9044a639-647e-4f0c-918c-292e75ba307e)

- edit file .env

```
UPLOAD_LOCATION=/data-immich2 ## ganti bagian ini
```

![image](https://github.com/galihtw04/photos-immch/assets/96242740/be0a3b02-9490-4a65-b5be-36787e3af5a0)

  - ekstrak 
```
tar -xvf data-immich.tar
mv data-immich /data-immich2
```

![image](https://github.com/galihtw04/photos-immch/assets/96242740/737a94e6-f598-4161-ab1e-b54107d9a705)

- create container

```
docker compose pull
docker compose create
docker start immich_postgres
sleep 25
```

![image](https://github.com/galihtw04/photos-immch/assets/96242740/a17a2aac-74a5-4d74-a3f6-dfce09e66bdc)

 - restore database
```
gunzip < "./dump.sql.gz" | docker exec -i immich_postgres psql -U postgres -d immich
```

  - start all container
```
docker compose up -d
```

![image](https://github.com/galihtw04/photos-immch/assets/96242740/5850c22f-8e58-45fb-9d94-5db54c80c602)

 - check akses
> login menggunakan email dan password pada sebelumnya.

![image](https://github.com/galihtw04/photos-immch/assets/96242740/c08d11ab-ebe3-4cb5-b37c-f60ed888ff36)

