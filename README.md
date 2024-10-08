# Bases de Datos Distribuidas Mongo Sharding

## Instalacion
1. Instalar Docker Desktop
2. Instalar NodeJS
3. Instalar Visual Studio Code o cualquier otro editor de codigo

## Arquitectura de la base de datos:

![DB_Architecture.png](https://raw.githubusercontent.com/AngelaGS02/Distributed_Databases_MongoSharding/main/misc/DB_Arquitecture.jpg)

# Paso a paso para crear la base de datos distribuida para el proyecto de patrimonio indígena

**Nombre de la base de datos**: indigenous_heritage

Tendremos:
- **Router**: `caucel` con el puerto 27017 dentro de un grupo llamado "router-ixora" con el puerto 27017
- **Config Servers**:
  - `tigrillo` and `puma` con el puerto 27019 dentro de un grupo llamado "config-hibisco" con el puerto 27019
- **Shards**:
  - `jaguar`, `manigordo`, `yaguarundi`, and `margay` dentro de un grupo llamado "shard-tabacon" con el puerto 27018

## 1. Hacer pull de Mongo, Crear Volumes, y Crear la Red

```bash
docker pull mongo:4.4
mkdir mongoContainer
cd mongoContainer

docker network create --driver bridge --subnet 10.0.0.32/28 cr_network
```

## 2. Crear los Shard Servers

```bash
docker volume create vol_jaguar
docker volume create vol_manigordo
docker volume create vol_yaguarundi
docker volume create vol_margay

docker run -d --net cr_network -v vol_jaguar:/data/db --ip 10.0.0.36 --name jaguar mongo:4.4 mongod --port 27018 --bind_ip_all --shardsvr --replSet "rep1" --dbpath /data/db
docker run -d --net cr_network -v vol_manigordo:/data/db --ip 10.0.0.37 --name manigordo mongo:4.4 mongod --port 27018 --bind_ip_all --shardsvr --replSet "rep1" --dbpath /data/db

docker run -d --net cr_network -v vol_yaguarundi:/data/db --ip 10.0.0.38 --name yaguarundi mongo:4.4 mongod --port 27018 --bind_ip_all --shardsvr --replSet "rep2" --dbpath /data/db
docker run -d --net cr_network -v vol_margay:/data/db --ip 10.0.0.39 --name margay mongo:4.4 mongod --port 27018 --bind_ip_all --shardsvr --replSet "rep2" --dbpath /data/db



```

### Inicializar la Replica para los Shards

```bash
docker exec -it jaguar mongo --port 27018

rs.initiate({
  _id: "rep1",
  members: [
    { _id: 0, host: "10.0.0.36:27018" },
    { _id: 1, host: "10.0.0.37:27018" }
  ]
});

cfg = rs.conf();
cfg.members[0].priority = 2;
cfg.members[1].priority = 0.5;
rs.reconfig(cfg);
```

### Habilitar las lecturas para los secundarios (esclavos)

```bash
rs.secondaryOk();
```

### Inicializar la Replica para yaguarundi

```bash
docker exec -it yaguarundi mongo --port 27018

rs.initiate({
  _id: "rep2",
  members: [
    { _id: 0, host: "10.0.0.38:27018" },
    { _id: 1, host: "10.0.0.39:27018" }
  ]
});

cfg = rs.conf();
cfg.members[0].priority = 2;
cfg.members[1].priority = 0.5;
rs.reconfig(cfg);

```

### Habilitar las lecturas para los secundarios (esclavos)

```bash
rs.secondaryOk();
```

### Asignar Arbitros para cada Replica

```bash
docker run -d --net cr_network --ip 10.0.0.40 --name arbiter_jaguar mongo:4.4 mongod --port 27018 --replSet "rep1" --dbpath /data/db
docker run -d --net cr_network --ip 10.0.0.41 --name arbiter_yaguarundi mongo:4.4 mongod --port 27018 --replSet "rep2" --dbpath /data/db
```

### Agregar Arbitro para jaguar

```bash
docker exec -it jaguar mongo --port 27018
rs.addArb("10.0.0.40:27018");
rs.status();
```

### Agregar Arbitro para yaguarundi

```bash
docker exec -it yaguarundi mongo --port 27018
rs.addArb("10.0.0.41:27018");
rs.status();
```

## 3. Crear los Config Servers

```bash

docker volume create vol_cfg_tigrillo
docker volume create vol_cfg_puma

docker run -d --net cr_network -v vol_cfg_tigrillo:/data/configdb --ip 10.0.0.34 --name cfg_tigrillo mongo:4.4 mongod --port 27019 --bind_ip_all --configsvr --replSet "repcfgregister" --dbpath /data/configdb
docker run -d --net cr_network -v vol_cfg_puma:/data/configdb --ip 10.0.0.35 --name cfg_puma mongo:4.4 mongod --port 27019 --bind_ip_all --configsvr --replSet "repcfgregister" --dbpath /data/configdb

```
### Entrar al Config Server de tigrillo

```bash
docker exec -it cfg_tigrillo mongo --port 27019

```

### Inicializar el Replica para los Config Servers

```bash
rs.initiate({
  _id: "repcfgregister",
  configsvr: true,
  members: [
    { _id: 0, host: "10.0.0.34:27019" },
    { _id: 1, host: "10.0.0.35:27019" }
  ]
});

cfg = rs.conf();
cfg.members[0].priority = 2;
cfg.members[1].priority = 0.5;
rs.reconfig(cfg);
```

### Habilitar las lecturas para los secundarios (esclavos)

```bash
rs.secondaryOk();
```



## 4. Crear el Router

```bash
docker run -d -p 27017:27017 --net cr_network --ip 10.0.0.42 --name router_caucel mongo:4.4 mongos --port 27017 --configdb repcfgregister/10.0.0.34:27019,10.0.0.35:27019 --bind_ip_all
docker exec -it router_caucel mongo --port 27017

db.adminCommand({
  "setDefaultRWConcern" : 1,
  "defaultWriteConcern" : {
    "w" : 1
  }
});
```


## 5. Iniciar y detener los servidores en orden

```bash
docker start jaguar manigordo yaguarundi margay
docker start arbiter_jaguar arbiter_yaguarundi
docker start cfg_tigrillo cfg_puma
docker start router_caucel
```

### Detener los servidores

```bash
docker stop router_caucel
docker stop arbiter_jaguar arbiter_yaguarundi
docker stop jaguar manigordo yaguarundi margay
docker stop cfg_tigrillo cfg_puma
```


## 6. Documentos de MongoDB

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

## 7. Agregar Shards y Tags

### Agregar Shards

```bash
use indigenous_heritage
sh.addShard("rep1/10.0.0.36:27018");
sh.addShard("rep2/10.0.0.38:27018");
sh.enableSharding("indigenous_heritage");
```

### Agregar Tags de Shard

```bash
sh.addShardTag("rep1", "puntarenas_sj_cartago_limon");
sh.addShardTag("rep2", "heredia_alajuela_guanacaste");

```

### Agregar Rangos de Tags -> Para recetas:



```bash
db.recipes.createIndex({ province: 1 })
sh.shardCollection("indigenous_heritage.recipes", { province: 1 })
sh.addTagRange("indigenous_heritage.recipes", { province: "San Jose" }, { province: "San Jose999"}, "puntarenas_sj_cartago_limon");
sh.addTagRange("indigenous_heritage.recipes", { province: "Cartago" }, { province: "Cartago999"}, "puntarenas_sj_cartago_limon");
sh.addTagRange("indigenous_heritage.recipes", { province: "Limon" }, { province: "Limon999"}, "puntarenas_sj_cartago_limon");
sh.addTagRange("indigenous_heritage.recipes", { province: "Puntarenas" }, { province: "Puntarenas999"}, "puntarenas_sj_cartago_limon");
sh.addTagRange("indigenous_heritage.recipes", { province: "Alajuela" }, { province: "Alajuela999"}, "heredia_alajuela_guanacaste");
sh.addTagRange("indigenous_heritage.recipes", { province: "Heredia" }, { province: "Heredia999"}, "heredia_alajuela_guanacaste");
sh.addTagRange("indigenous_heritage.recipes", { province: "Guanacaste" }, { province: "Guanacaste999"}, "heredia_alajuela_guanacaste");
db.recipes.getShardDistribution()
```

### Agregar Tags de Shard para grupos

```bash
sh.addTagRange("indigenous_heritage.indigenous_groups.location.province", { province: "San Jose" }, { province: "San Jose999"}, "puntarenas_sj_cartago_limon");
sh.addTagRange("indigenous_heritage.indigenous_groups.location.province", { province: "Cartago" }, { province: "Cartago999"}, "puntarenas_sj_cartago_limon");
sh.addTagRange("indigenous_heritage.indigenous_groups.location.province", { province: "Limon" }, { province: "Limon999"}, "puntarenas_sj_cartago_limon");
sh.addTagRange("indigenous_heritage.indigenous_groups.location.province", { province: "Puntarenas" }, { province: "Puntarenas999"}, "puntarenas_sj_cartago_limon");
sh.addTagRange("indigenous_heritage.indigenous_groups.location.province", { province: "Alajuela" }, { province: "Alajuela999"}, "heredia_alajuela_guanacaste");
sh.addTagRange("indigenous_heritage.indigenous_groups.location.province", { province: "Heredia" }, { province: "Heredia999"}, "heredia_alajuela_guanacaste");
sh.addTagRange("indigenous_heritage.indigenous_groups.location.province", { province: "Guanacaste" }, { province: "Guanacaste999"}, "heredia_alajuela_guanacaste");

```

## 8. Habilitar Sharding para la base de datos

```bash
sh.enableSharding("indigenous_heritage");
```

## 9. Cargar Datos desde archivos JSON y XLSX

### Acceder a la carpeta con los datos de prueba

```bash
cd ..
cd client
cd app
cd data
```

### Copiar los archivos a la carpeta de la base de datos

```bash
docker cp recetas.json router_caucel:/recipes.json
docker cp ingredientes.json router_caucel:/ingredientes.json
docker cp grupos.json router_caucel:/groups.json
docker cp festividad.csv router_caucel:/festividad.csv 
```

### Importar los datos a la base de datos

```bash
docker exec -it router_caucel mongoimport --db indigenous_heritage --collection recipes --file /recipes.json --jsonArray
docker exec -it router_caucel mongoimport --db indigenous_heritage --collection indigenous_ingredients --file /ingredientes.json --jsonArray
docker exec -it router_caucel mongoimport --db indigenous_heritage --collection indigenous_groups --file /groups.json --jsonArray
docker exec -it router_caucel mongoimport --db indigenous_heritage --collection festivities --file /festividad.csv --type csv --headerline
```

### Verificar la importacion

```bash
docker exec -it router_caucel mongo --port 27017
use indigenous_heritage
db.recipes.getShardDistribution()
exit
```
### Accedemos a un nodo de la replica 1 para verificar los datos (San Jose, Cartago, Limon, Puntarenas)

```bash
docker exec -it jaguar mongo --port 27018
use indigenous_heritage
db.recipes.find().pretty()
exit
```

### Accedemos a otro nodo de la replica 2 para verificar los datos (Heredia, Alajuela, Guanacaste)

```bash
docker exec -it yaguarundi mongo --port 27018
use indigenous_heritage
db.recipes.find().pretty()
exit
```

## 10. Probamos la tolerancia a fallos

### Detenemos un nodo de la replica 1 (jaguar) y verificamos que el segundo nodo (manigordo) se convierta en PRIMARY

```bash
docker stop jaguar
docker exec -it manigordo mongo --port 27018
use indigenous_heritage
db.recipes.find().pretty()
exit
```

### Detenemos el nodo PRIMARY de la replica 2 (yaguarundi) y verificamos que el segundo nodo (margay) se convierta en PRIMARY

```bash
docker stop yaguarundi
docker exec -it margay mongo --port 27018
use indigenous_heritage
db.recipes.find().pretty()
exit
```
### Volvemos a iniciar los nodos detenidos y verificamos que regresen a su estado PRIMARY

```bash
docker start jaguar yaguarundi
docker exec -it yaguarundi mongo --port 27018
use indigenous_heritage
db.recipes.find().pretty()
exit

docker exec -it jaguar mongo --port 27018
use indigenous_heritage
db.recipes.find().pretty()
exit
```

# Conclusión

Con esta guia pudimos crear una base de datos distribuida con 4 shards, 2 config servers y 1 router, ademas de poder agregarle datos y verificar la distribucion y tolerancia a fallos.

Estos servicios usualmente se usan en conjunto con Mongo Express para tener una interfaz grafica y administrar la base de datos de una manera mas sencilla.
En este repositorio se crea una aplicacion web con NexJS y MongoDB, que usa estas bases de datos distribuidas, permitiendo diferentes operaciones CRUD a las colecciones de la base de datos.

## Ver README.md en la carpeta client para mas detalles de la aplicacion.










