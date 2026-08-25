# How to Build an Apptainer Container That Assembles a Mongo Database

### Apptainer Setup
 [Apptainer installation page](https://apptainer.org/user-docs/master/quick_start.html#installation)

## Step 1: Creating the Image File

It is assumed that apptainer now works on your system, meaning you can build image files and run them as containers. If apptainer doesn't work properly, please refer to the link above. 

To create our image file (.sif), we will need to define a .def file. The .def file is a text file that tells apptainer how to build the image file. More information on .def file creation is on [the apptainer .def file documentation](https://apptainer.org/user-docs/master/definition_files.html).

### Explaining My .def File

My .def file is as follows: 
```
Bootstrap: docker
From: ubuntu:24.04

%files
    /mnt/e/Program-Files/Apptainer_Images/run_scripts/DB_test_run.sh /opt/mongodb/runfile.sh

%post
    export DEBIAN_FRONTEND=noninteractive

    apt-get update
    apt-get install -y --no-install-recommends \
        python3 \
        python3-pip \
        gnupg \
        curl \
        ca-certificates

    curl -fsSL https://pgp.mongodb.com/server-8.0.asc | \
        gpg --dearmor -o /usr/share/keyrings/mongodb-server-8.0.gpg

    echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" \
        > /etc/apt/sources.list.d/mongodb-org-8.0.list

    apt-get update
    apt-get install -y --no-install-recommends mongodb-org

    mkdir -p /shard_cluster
    chmod +x /opt/mongodb/runfile.sh

    apt-get clean
    rm -rf /var/lib/apt/lists/*

%environment
    export LC_ALL=C
    export MONGO_DATA_DIR=/data/db

%runscript
    exec /opt/mongodb/runfile.sh
```


At the top, I dictate that the image run on ubuntu version 24.04, and the infrastructure needed for running the OS is from Docker. 

### The %files Section
This section copies the local file at /mnt/e/Program-Files/Apptainer_Images/run_scripts/DB_test_run.sh onto the container at /opt/mongodb/runfile.sh, so the container can reference it later. This is a custom bash script that sets up our sharded cluster and database, but more on that later.

### The %post Section 
We start with the first line, which tells ubuntu to be silent when downloading packages. 

In lines 25 through 31, we download necessary libraries. Python will be necessary when testing the database, which is a future feature.

In lines 33 through 40, we download Mongo community edition as specified in [the Mongo documentation](https://www.mongodb.com/docs/manual/administration/install-community/?operating-system=linux&linux-distribution=ubuntu&linux-package=default).

On line 42, I create a mount point for accessing sharded cluster data that must be stored on the host. A mount point is a directory in the container that attaches to a directory on the host, which is specified later. We need this mount point because each shard (replica set) has one or more mongod processes running, and each mongod process needs the ability to write data to disk. Image files are read-only for the container. As of now, I have an external script that creates files on the host for each mongod instance that connects to the mount point above. However, creating a %setup section should allow me to create the necessary files on the host when running the container, which would be cleaner.

On line 43, I ensure that the runfile copied to the image file in the %files section is executable.

In lines 45 and 46, I clean up temporary files I no longer need. This is to keep the image file as compact as possible.

### The %environment Section



### The %runscript Section
I run the runfile copied into the image file in the %files section.



## Step 2: Creating the Runfile

My runfile is as follows (this part will be updated in the future. As for now, the [Mongo documentation](https://www.mongodb.com/docs/manual/) should help explain some of it:
```
#!/bin/bash

mkdir -p /shard_cluster/configshrd
mkdir -p /shard_cluster/shrdsvr_1
mkdir -p /shard_cluster/shrdsvr_2
mkdir -p /shard_cluster/shrdsvr_3
mkdir -p /shard_cluster/shrdsvr_4
mkdir -p /shard_cluster/shrdsvr_5
mkdir -p /shard_cluster/shrdsvr_6
mkdir -p /raw_data

wait_for_mongo() {
    local port="$1"

    echo "Waiting for MongoDB on port $port..."

    until mongosh \
        --port "$port" \
        --quiet \
        --eval 'db.adminCommand({ping: 1}).ok' \
        2>/dev/null | grep -q '^1$'   # asks Mongo if it's working, redirects stderr into linux's garbage can, then extracts Mongo's response and checks if it's *only* working and nothing else.
    do
        sleep 1
    done

    echo "MongoDB on port $port is ready."
}

rs_initialize() {
    local port="$1"
    local id="$2"

    mongosh --port "$port" --eval "
        rs.initiate({
            _id: '$id',
            members: [
                { _id: 0, host: '127.0.0.1:$port' }
            ]
        })
    "
}

echo "Starting MongoDB cluster..."


echo "Starting MongoDB..."

mongod \
    --dbpath /shard_cluster/configshrd \
    --configsvr \
    --replSet configsvr \
    --bind_ip 127.0.0.1 \
    --port 27050 \
    --fork \
    --logpath /tmp/configsvr.log

wait_for_mongo 27050

rs_initialize 27050 configsvr

mongod \
    --shardsvr \
    --replSet shrdsvr_1 \
    --port 27051 \
    --dbpath /shard_cluster/shrdsvr_1 \
    --fork \
    --logpath /tmp/shrdsvr_1.log

mongod \
    --shardsvr \
    --replSet shrdsvr_2 \
    --port 27052 \
    --dbpath /shard_cluster/shrdsvr_2 \
    --fork \
    --logpath /tmp/shrdsvr_2.log

mongod \
    --shardsvr \
    --replSet shrdsvr_3 \
    --port 27053 \
    --dbpath /shard_cluster/shrdsvr_3 \
    --fork \
    --logpath /tmp/shrdsvr_3.log

mongod \
    --shardsvr \
    --replSet shrdsvr_4 \
    --port 27054 \
    --dbpath /shard_cluster/shrdsvr_4 \
    --fork \
    --logpath /tmp/shrdsvr_4.log

mongod \
    --shardsvr \
    --replSet shrdsvr_5 \
    --port 27055 \
    --dbpath /shard_cluster/shrdsvr_5 \
    --fork \
    --logpath /tmp/shrdsvr_5.log

mongod \
    --shardsvr \
    --replSet shrdsvr_6 \
    --port 27056 \
    --dbpath /shard_cluster/shrdsvr_6 \
    --fork \
    --logpath /tmp/shrdsvr_6.log

wait_for_mongo 27051
wait_for_mongo 27052
wait_for_mongo 27053
wait_for_mongo 27054
wait_for_mongo 27055
wait_for_mongo 27056

rs_initialize 27051 shrdsvr_1
rs_initialize 27052 shrdsvr_2
rs_initialize 27053 shrdsvr_3
rs_initialize 27054 shrdsvr_4
rs_initialize 27055 shrdsvr_5
rs_initialize 27056 shrdsvr_6

mongos \
    --configdb configsvr/localhost:27050 \
    --port 27060 \
    --bind_ip 127.0.0.1 \
    --fork \
    --logpath /tmp/mongos.log

wait_for_mongo 27060

mongosh --port 27060 --eval '
    sh.addShard("shrdsvr_1/127.0.0.1:27051");
    sh.addShard("shrdsvr_2/127.0.0.1:27052");
    sh.addShard("shrdsvr_3/127.0.0.1:27053");
    sh.addShard("shrdsvr_4/127.0.0.1:27054");
    sh.addShard("shrdsvr_5/127.0.0.1:27055");
    sh.addShard("shrdsvr_6/127.0.0.1:27056");

    db = db.getSiblingDB("testDB");
    db.testCollection.createIndex({_id: "hashed"});
    sh.shardCollection("testDB.testCollection", {_id: "hashed"});
'

mongorestore --host 127.0.0.1 --port 27060 \
    --archive="/raw_data/boom_no_cutouts.archive" \
    # --nsFrom="boom.DESI_DR1" \
    # --nsTo="testDB.testCollection"

mongosh --port 27060 --quiet --eval 'sh.status()'
```





## Step 3: Running the Container

### To build the image file:

apptainer build (designated location for image file) (path to .def file)

### To run the container:

apptainer run \
    --bind (designated path to directory on host for Mongod processes):/shard_cluster \
    --bind (path to Mongo .archive file, or jsonl file):/raw_data \
    (path to image file) 

 


