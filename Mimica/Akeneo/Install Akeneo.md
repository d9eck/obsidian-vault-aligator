Preinstallation
Create directories
```
sudo mkdir -p /data/volumes/akeneo/{pim-root,mysql,elasticsearch,minio}

# Akeneo app files live as www-data in the containers
sudo chown -R 33:33 /data/volumes/akeneo/pim-root

# MySQL official image uses uid/gid 999
sudo chown -R 999:999 /data/volumes/akeneo/mysql

# Elasticsearch official image uses uid 1000 (group 1000 typically)
sudo chown -R 1000:1000 /data/volumes/akeneo/elasticsearch

# MinIO runs as a non-root user as well; 1000:1000 is safe for hostPath
sudo chown -R 1000:1000 /data/volumes/akeneo/minio
```

