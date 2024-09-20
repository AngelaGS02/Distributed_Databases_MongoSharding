# Distributed Databases MongoSharding

## Installation
1. Install Docker Desktop
2. Install NodeJS'
3. Install Visual Studio Code or any other Code Editor

## Database Architecture:

![DB_Architecture.png](https://raw.githubusercontent.com/AngelaGS02/Distributed_Databases_MongoSharding/main/misc/DB_Arquitecture.jpg)

# Step-by-Step Guide to Create the Distributed Database for the Indigenous Heritage Project

**Database Name**: indigenous_heritage

We will have:
- **Router**: `caucel` with port 27017 inside a group called "router-ixora" with port 27017
- **Config Servers**:
  - `tigrillo` and `puma` with port 27019 inside a group called "config-hibisco" with port 27019
- **Shards**:
  - `jaguar`, `manigordo`, `yaguarundi`, and `margay` inside a group called "shard-tabacon" with port 27018

## 1. Pull Mongo, Create Volumes, and Network

```bash
docker pull mongo:4.4
mkdir mongoContainer
cd mongoContainer

docker volume create vol_cfg_tigrillo
docker volume create vol_cfg_puma

docker network create --driver bridge --subnet 10.0.0.32/28 cr_network
```

## 2. Create the Mongo Cluster (Sharding and Replication)

```bash
docker run -d --net cr_network -v vol_cfg_tigrillo:/data/configdb --ip 10.0.0.34 --name cfg_tigrillo mongo:4.4 mongod --port 27019 --configsvr --replSet "repcfgregister" --dbpath /data/configdb
docker run -d --net cr_network -v vol_cfg_puma:/data/configdb --ip 10.0.0.35 --name cfg_puma mongo:4.4 mongod --port 27019 --configsvr --replSet "repcfgregister" --dbpath /data/configdb
```
### Enter tigrillo Config Server

```bash
docker exec -it cfg_tigrillo mongo --port 27019
```

### Initialize Replica for Config Servers

```bash
rs.initiate({
  _id: "repcfgregister",
  configsvr: true,
  members: [
    { _id: 0, host: "10.0.0.34:27019" },
    { _id: 1, host: "10.0.0.35:27019" }
  ]
});
```

### Enable Reads for Secondary (Slave)

```bash
rs.secondaryOk();
```

## 3. Create Shard Servers

```bash
docker volume create vol_jaguar
docker volume create vol_manigordo
docker volume create vol_yaguarundi
docker volume create vol_margay

docker run -d --net cr_network -v vol_jaguar:/data/db --ip 10.0.0.36 --name jaguar mongo:4.4 mongod --port 27018 --shardsvr --replSet "rep1" --dbpath /data/db
docker run -d --net cr_network -v vol_manigordo:/data/db --ip 10.0.0.37 --name manigordo mongo:4.4 mongod --port 27018 --shardsvr --replSet "rep1" --dbpath /data/db

docker run -d --net cr_network -v vol_yaguarundi:/data/db --ip 10.0.0.38 --name yaguarundi mongo:4.4 mongod --port 27018 --shardsvr --replSet "rep2" --dbpath /data/db
docker run -d --net cr_network -v vol_margay:/data/db --ip 10.0.0.39 --name margay mongo:4.4 mongod --port 27018 --shardsvr --replSet "rep2" --dbpath /data/db

```

### Initialize Replica for Shards

```bash
docker exec -it jaguar mongo --port 27018

rs.initiate({
  _id: "rep1",
  members: [
    { _id: 0, host: "10.0.0.36:27018" },
    { _id: 1, host: "10.0.0.37:27018" }
  ]
});
```

### Enable Reads for Secondary (Slave)

```bash
rs.secondaryOk();
```

### Initialize Replica for yaguarundi

```bash
docker exec -it yaguarundi mongo --port 27018

rs.initiate({
  _id: "rep2",
  members: [
    { _id: 0, host: "10.0.0.38:27018" },
    { _id: 1, host: "10.0.0.39:27018" }
  ]
});

```

### Enable Reads for Secondary (Slave)

```bash
rs.secondaryOk();
```

### Set Arbiters for Each Replica

```bash
docker run -d --net cr_network --ip 10.0.0.40 --name arbiter_jaguar mongo:4.4 mongod --port 27018 --replSet "rep1" --dbpath /data/db
docker run -d --net cr_network --ip 10.0.0.41 --name arbiter_yaguarundi mongo:4.4 mongod --port 27018 --replSet "rep2" --dbpath /data/db
```

### Add Arbiter for jaguar

```bash
docker exec -it jaguar mongo --port 27018
rs.addArb("10.0.0.40:27018");
rs.status();
```

### Add Arbiter for yaguarundi

```bash
docker exec -it yaguarundi mongo --port 27018
rs.addArb("10.0.0.41:27018");
rs.status();
```

## 4. Create Router

```bash
docker run -d -p 27017:27017 --net cr_network --ip 10.0.0.42 --name router_caucel mongo:4.4 mongos --port 27017 --configdb repcfgregister/10.0.0.34:27019,10.0.0.35:27019
docker exec -it router_caucel mongo --port 27017
```

## 5. Start and Stop Servers in Order

```bash
docker start jaguar manigordo yaguarundi margay
docker start arbiter_jaguar arbiter_yaguarundi
docker start cfg_tigrillo cfg_puma
docker start router_caucel
```

### Stop Servers

```bash
docker stop router_caucel
docker stop arbiter_jaguar arbiter_yaguarundi
docker stop jaguar manigordo yaguarundi margay
docker stop cfg_tigrillo cfg_puma
```

## 6. MongoDB Documents

### Recetas

```javascript
Recetas = {
  province: Number,
  recipe_name: String,
  ingredients: [Ingredient],
  preparation_steps: [String],
  preparation_time: String,
  occasion: String,
  who_prepares: String,
  used_for_festivities: boolean
};
```

### Ingredientes

```javascript
Ingredientes = {
  name_spanish: String,
  names_indigenous_languages: String,
  production_location: String,
  exists_today: boolean,
  consumption_by_group: String
};
```

### Grupos

```javascript
Grupos = {
  name: String,
  location: String,
  population: { year: Number, number: Number },
  languages_spoken: [String],
  social_structure: {
    clans: [String],
    leadership: String,
    cultural_practices: String
  }
};
```

### Festividades

```javascript
Festividades = {
  Nombre_Original: String,
  Fecha: Date,
  Actividades: [String],
  Quien_Puede_Asistir: [String],
  Implicaciones: String
};
```

## 7. Add Shards and Tags

### Add Shards

```bash
sh.addShard("rep1/10.0.0.36:27018");
sh.addShard("rep2/10.0.0.38:27018");
```

### Add Shard Tags

```bash
sh.addShardTag("rep1", "puntarenas_sj_cartago_limon");
sh.addShardTag("rep2", "heredia_alajuela_guanacaste");

```

### Add Tag Ranges

```bash

sh.addTagRange("<database>.<collection>", min, max, "tag");

sh.addTagRange("indigenous_heritage.recipes", { province: 1 }, { province: 4 }, "puntarenas_sj_cartago_limon");
sh.addTagRange("indigenous_heritage.recipes", { province: 5 }, { province: 7 }, "heredia_alajuela_guanacaste");
```

## 8. Enable Sharding for Database

```bash
sh.enableSharding("indigenous_heritage");
```

## Load Data from JSON and XLSX Files


mongoimport --db indigenous_heritage --collection recipes --file recipes.json

// Load csv files

mongoimport --db indigenous_heritage --collection ingredients --type csv --file ingredients.csv --headerline
