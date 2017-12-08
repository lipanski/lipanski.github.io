## Notes on installing TimescaleDB

```sh
sudo add-apt-repository ppa:timescale/timescaledb-ppa
sudo apt-get update
sudo apt install timescaledb
```

Edit `​/etc/postgresql/9.6/main/postgresql.conf`:

```
shared_preload_libraries = 'timescaledb'
```

There's more...
