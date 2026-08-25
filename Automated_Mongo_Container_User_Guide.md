# How to Build an Apptainer Container That Assembles a Mongo Database

### Apptainer Setup
 [Apptainer installation page](https://apptainer.org/user-docs/master/quick_start.html#installation)

## Step 1

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




