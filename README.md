## Run PDAL in a Docker Container

This is a tutorial to demonstrate how to containerize and run PDAL. [PDAL](https://pdal.io/en/2.6.0/) is a stand-alone software package that can analyze and manipulate point cloud data files such as .las and .laz. In this tutorial, we will convert a LiDAR .laz file into a [Cloud-optimized Point Cloud format (.copc.laz)](https://www.gillanscience.com/cloud-native-geospatial/copc/). 

### 1. Clone this repository to your local machine

`git clone https://github.com/jeffgillan/pdal_copc.git`

### 2. Change directories into the newly cloned repository

`cd pdal_copc`

### 3. Run the Container

`docker run -v $(pwd):/data jeffgillan/pdal_copc:1.0`

Your if everything worked correctly, you should have a new file `tree.copc.laz` in your present working directory.
___
</br>

## How to Build this Docker Container

### 1. PDAL Pipeline

Analyzing pointclouds in PDAL requires users to specify processing steps within a json file. PDAL uses the term ['pipeline'](https://pdal.io/en/2.6.0/pipeline.html) to describe this json file.

#### Create an empty json file 

`touch copc.json`
</br>

#### Open the pipeline file

`nano copc.json`

</br>

#### Write this info in the pipeline file. It will convert .las & .laz files to cloud optimized point clouds (.copc.laz)
```
[
    "file.las",
    "output.copc.laz"
]
```
</br>

### 2. Create a shell script

The shell script will loop through a directory (within the container) and find all .laz and .las files and then convert them to copc.laz. The shell script references the json file (pipeline), so the path to the json needs to be specified within the shell script. 

`touch pdal_copc.sh`

#### Edit the shell script
`nano pdal_copc.sh`

```
#!/bin/bash

#Define the pipeline JSON file
pipeline="/app/copc.json" #use this path if in a container
#pipeline="./copc.json"  #use this path if you are running the shell script on your local conda environment

### Loop over LAS/LAZ files in a directory. If you are running this shell script in a container, then the script is looking for data in the `/data` in the container directory structure. 
find /data -type f \( -name "*.las" -o -name "*.laz" \) -print0 | while IFS= read -r -d '' file; do
    # Get the file extension
    extension="${file##*.}"

    # Run the pipeline with the appropriate reader based on the file extension
    if [[ "$extension" == "las" ]]; then
        pdal pipeline -i "$pipeline" --readers.las.filename="$file" --writers.copc.filename="${file%.las}.copc.laz"
    elif [[ "$extension" == "laz" ]]; then
        pdal pipeline -i "$pipeline" --readers.las.filename="$file" --writers.copc.filename="${file%.laz}.copc.laz"
    fi
done
```

### 3. Create a Dockerfile 

You are creating a Docker image that includes a PDAL base image, the json pipeline file, and the shell script. 

#### Create an empty Dockerfile

`touch Dockerfile`

#### Edit the Dockerfile

`nano Dockerfile`

```
FROM pdal/pdal:sha-597ab2df

WORKDIR /app

COPY pdal_copc.sh /app/pdal_copc.sh

COPY copc.json /app/copc.json

RUN chmod +x pdal_copc.sh

ENTRYPOINT ["/app/pdal_copc.sh"]
```

The following is happening in the Dockerfile:

A PDAL base image is being pulled from Dockerhub

I set the working directory of the container to `/app`

I copy in the shell script to the path `/app`

I copy in the pipeline json file to the path `/app`

I run `chmod +x` on the shell script to give everyone permissions

The entrypoint is where the container starts. I want it to start with the shell script.

### 4. Build the docker image
You are telling it to build an image with the name 'jeffgillan/pdal_copc' with the tag '1.0'. You are building from the Dockerfile in the current working directory '.'

`docker build -t jeffgillan/pdal_copc:1.0 .`

### 5. Run the container 
You are mounting a local volume (-v) directory to the container (`/data`). This local directory should have all of the point clouds files you want to convert. `$(pwd)` is telling it that the point clouds are in the current working directory. Alternatively, you could specify the point clouds are locating in any local directory.

`docker run -v $(pwd):/data jeffgillan/pdal_copc:1.0`


### 6. Outputs

The tool should output `.copc.laz` files to the same directory where the input point clouds were storesd. It is slow and might take a while.   

### 7. Upload Image to Docker Hub

`docker push jeffgillan/pdal_copc:1.0`

</br>
____

## Running Pdal from local conda environment

Create a new conda environment

`conda create --yes --name pdal_copc --channel conda-forge pdal`


In your conda environment (conda activate pdal_copc) run the following commands to run the shell script

```
chmod +x pdal_copc.sh
./pdal_copc.sh
```


