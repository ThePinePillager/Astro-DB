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




