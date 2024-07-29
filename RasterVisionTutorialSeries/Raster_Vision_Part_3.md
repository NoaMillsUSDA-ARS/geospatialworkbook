---
title: "Raster Vision Tutorial Series Part 3: Constructing and Exploring the Apptainer Image"
layout: single
author: Noa Mills
author_profile: true
header:
  overlay_color: "444444"
  overlay_image: /assets/images/margaret-weir-GZyjbLNOaFg-unsplash_dark.jpg
---

# Semantic Segmentation of Aerial Imagery with Raster Vision 
## Part 3: Constructing and Exploring the Apptainer Image

This tutorial series walks through an example of using [Raster Vision](https://rastervision.io/) to train a deep learning model to identify buildings in satellite imagery.<br>

*Primary Libraries and Tools*:

|Name|Description|Link|
|-|-|-|
| `Raster Vision ` | Library and framework for geospatial semantic segmentation, object detection, and chip classification in python| https://rastervision.io/ |
| `Apptainer` | Containerization software that allows for transportable and reproducible software | https://apptainer.org/ |
| `pandas` | Python library supporting dataframes and other datatypes for data analysis and manipulation | https://pandas.pydata.org/ |
| `geopandas` | Python library that extends pandas to support geospatial vector data and spatial operations | https://geopandas.org/en/stable/ |
| `rioxarray` | Python library supporting data structures and operations for geospatial raster data | https://github.com/corteva/rioxarray |
| `pathlib` | A Python library for handling files and paths in the filesystem | https://docs.python.org/3/library/pathlib.html |

*Prerequisites*:
  * Basic understanding of navigating the Linux command line, including navigating among directories and editing text files
  * Basic python skills, including an understanding of object-oriented programming, function calls, and basic data types
  * Basic understanding of shell scripts and job scheduling with SLURM for running code on Atlas
  * A SCINet account for running this tutorial on Atlas
  * **Completion of tutorial parts 1 and 2 of this series**

*Tutorials in this Series*:
  * 1\. **Tutorial Setup on SCINet**
  * 2\. **Overview of Deep Learning for Imagery and the Raster Vision Pipeline**
  * 3\. **Constructing and Exploring the Apptainer Image <span style="color: red;">_(You are here)_</span>**
  * 4\. **Exploring the Dataset and Problem Space**
  * 5\. **Overview of Raster Vision Model Configuration and Setup**
  * 6\. **Breakdown of Raster Vision Code Version 1**
  * 7\. **Evaluating Training Performance and Visualizing Predictions**
  * 8\. **Modifying Model Configuration - Hyperparameter Tuning**

## Constructing and Exploring the Apptainer Image

#### Users who are not familiar with containerization are strongly encouraged to go through [this tutorial](https://hsf-training.github.io/hsf-training-singularity-webpage/index.html). <br>

##### Terminology note: Apptainer vs Singularity
Apptainer used to be called Singularity, and the name changed when the Singularity project [moved to the Linux Foundation](https://apptainer.org/news/community-announcement-20211130/) in 2021. The two softwares work the same, just with different terminology. For example, you now use the `apptainer` command in place of the `singularity` command. You see in this tutorial a few instances in which the "singularity" terminology still persists, even when interacting with the Apptainer module. For example, Apptainer/Singularity image files, even those built with Apptainer, still have the extension ".sif", which stands for Singularity Image File.

### 1. Containerization Background and Setup
One of the most difficult aspects of software development is setting up the computing environment - ensuring you are running your code with all the right software configurations set and dependency versions installed. You may build an application on your machine, but struggle to get it to work the same way on a different machine because of differing software installations and configurations. Containerization is used to prevent dependency issues and improve the portability of code. Containers are collections of code along with all the needed dependencies. They can be easily moved from one machine to another, and perform identically regardless of where they are running, and users don't need to install all of the dependencies and ensure they are all the right versions - they just need the image. <br>

##### Terminology note: an *image* is a snapshot of a computing environment, like a blueprint for a container. A *container* is an isolated computing environment built from the instructions in the image. Containers are running instances of images.<br>

The developers of Raster Vision publish the Raster Vision software as Docker images to simplify the process of running the Raster Vision pipeline. New versions of Raster Vision are released as Docker images [here](https://quay.io/repository/azavea/raster-vision?tab=tags). Docker and Apptainer are two different containerization platforms, each with their own pros and cons. Docker is a popular containerization tool, however it requires root access to run and therefore can't be used on an HPC like Atlas. Apptainer (formerly Singularity), on the other hand, is designed to not require root access so it can be used on an HPC system. Thankfully, we can create an Apptainer image out of a Docker image, so we can run the Raster Vision code on Atlas. In the following instructions, we will use build an Apptainer image out of the Raster Vision Docker image. <br><br>
First, ensure that the variables `$project_dir` and `$project_name` are available. If you have started a new Jupyter session since creating these variables in tutorial 1 of this series, then you will need to create them again. Check to see if they are available by running:<br>
`echo $project_dir`<br>
`echo $project_name`<br>
##### If the project directory and project name do not appear, then return to the tutorial setup instructions in Part 1 of the series to create these variables before proceeding. <br>

By default, apptainer will cache all downloaded images to `$HOME/.apptainer` so if the user deletes an image and attempts to re-download the same version, the image will be pulled from the local cache instead of a remote repository. This is a useful feature to decrease network demand, however Atlas users have limited space in their home directories and the apptainer cache can quickly fill up the limited space. The SCINet office recommends configuring the cache directory as follows to avoid filling up your home directory:<br><br>
`export APPTAINER_CACHEDIR=$TMPDIR`<br>
`export APPTAINER_TMPDIR=$TMPDIR`<br><br>

Next, we will navigate to the project directory and run a script to pull a Raster Vision image from the remote repository. Note that this will take a while to run, so we recommend continuing with the following reading while this code runs. <br><br>
`cd ${project_dir}/model` <br>
`sbatch --account=$project_name make_apptainer_img.sh` 

### 2. Apptainer File Systems
In addition to providing an isolated computing environment, apptainer containers also have their own file systems separate from the host system's file system. Directories in the host system are made available within the container's file system by _binding_ directories. For example, say you have a directory of data files on the host file system at `/project/example/data` that you would like to have access to within the container. You could make this directory available within the container by binding the directory `/project/example/data` to a directory in the container's file system, such as `/opt/data`. Then, when you start the container, you can navigate to `/opt/data` within the container and access the files in `/project/example/data` on the host system. If you modify files in the container in `/opt/data`, then these changes will also affect the host system at `/project/example/data`. This way, we can save files to the host system from within the container to access later. Note that the permissions you have on the host system will be identical to the permissions you have within the container, so you can't perform any actions to the host's file system within a container that you couldn't otherwise do outside of the container.<br><br>
Depending on the administrative configurations of the host system, certain directories in the host's file system are bound to directories in the container's file system by default. For example, it is common for the directory `$HOME` in the host's file system to be bound to the directory `/home` within the container, and for the working directory on the host system to be bound to a directory with the same name in the container. If you wish to bind additional directories, you can specify the directories you'd like to bind when you launch the container. We will discuss the specifics of how to bind directories later in section 4 of this tutorial after we discuss how to launch a container.

### 3. Launching an Apptainer Container

There are several apptainer commands that we can use to launch a apptainer container from an apptainer image file (.sif file). The most common commands are `shell`, `run`, and `exec`. Here is a quick overview of these three commands: <br>

`apptainer shell my_image.sif` will build the container and launch an interactive shell environment in the container. This is useful for exploring the container interactively, and for debugging. You can shut down the container with the `exit` command. We will use this command soon to explore the Raster Vision container.<br><br>
`apptainer run my_image.sif` will run the default _runscript_ within the my_image container. A _runscript_ is included within an apptainer image to specify the default behavior when we "run" a container. <br><br>
`apptainer exec my_image.sif command` allows us to run a specific command within the container, instead of the default behavior described in the runscript. For example, `apptainer exec my_image.sif python python_script.py` will execute the `python_script.py` within the container. <br>

### 4. Exploring the Raster Vision Container

Once the `make_apptainer_img.sh` script has completed running, you should see the file `raster-vision_pytorch-0.30.sif` in your `model/` directory. We will first explore the container as is, then we will bind a directory of data files from the host system to a directory within the container. First, load the apptainer module:<br>
`module load apptainer` <br><br>
Then, from your `model/` directory, run the command: <br>
`apptainer shell raster-vision_pytorch-0.30.sif` <br><br>
The container will take a minute to launch. Once it does, you will see your prompt changes to `Singularity >`. Next, run the commands: <br>
`pwd` <br>
`ls` <br><br>
You will see the `model/` directory that you launched the apptainer container from. <b>By default, apptainer binds the working directory on the host system to the same directory path in the container.</b> Here, we can see that apptainer has created the full path directory, `/path/to/your/model/`, and we can see all of our files from the `model/` directory within the container. If we modify these files within our container, then these changes will also be reflected on the host system. 

Next, run the commands: <br>
`cd /opt/src` <br>
`ls` <br><br>
Here we have the directory for the Raster Vision files within the container. We won't need to touch these files in order to run the pipeline, but this is where the code is that runs the pipeline. When new versions of Raster Vision are released, new containers are published with updated code in this directory. 

Next we will launch the container with our data directory bound to the container. To exit the container, run the command:<br>
`exit`<br><br>

To bind a directory to the container, we use the option `-B` or `--bind`, followed by our binding specifications in the format `/host/system/directory/:/container/directory/`. Our input data files are stored at `/reference/workshops/rastervision/input/`. Run the following command to launch the container with the `input/` directory on the host system bound to `/opt/data/input/` in the container. Note that if the directory we specify does not already exist in the container, it will be created. <br>
``apptainer shell -B /reference/workshops/rastervision/input/:/opt/data/input/ raster-vision_pytorch-0.30.sif`` <br>
`cd /opt/data/input` <br>
`ls` <br><br>
You should see three subdirectories: `train`, `val` and `test`. You can check out the contents as follows: <br>
`cd train`<br>
`ls | head -n 20` # List the first 20 lines of directory contents. <br><br>
Now we can access our data within our container!

#### Conclusion
You should now have a basic understanding of the apptainer image, and how we access files on the host system from within the container. In the next tutorial, we will explore the dataset we will use for this tutorial, the problem space, and why building a Raster Vision model is a good choice for our specific goals.