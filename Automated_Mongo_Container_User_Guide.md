# How to Build an Apptainer Container That Assembles a Mongo Database

### Apptainer Setup
 [Apptainer installation page](https://apptainer.org/user-docs/master/quick_start.html#installation)

### Step 1

It is assumed that apptainer now works on your system, meaning you can build image files and run them as containers. If apptainer doesn't work properly, please refer to the link above. 

To create our image file (.sif), we will need to define a .def file. The .def file is a text file that tells apptainer how to build the image file. More information on .def file creation is on [the apptainer .def file documentation](https://apptainer.org/user-docs/master/definition_files.html)
