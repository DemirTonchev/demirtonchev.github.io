---
layout: post
title: "Using docker and mamba without tears"
categories: MLeng
tags: mamba docker
author:
- Me
---

If you are here, you probably know what [conda](https://docs.conda.io/en/latest/) is (open-source package manager and environment manager). This means that you can use it to create_ virtual environments and _install_ packages within, also conda will take care of interdependencies of the packages. If a package is not available on conda channels you can always resolve to pip to install it.

So conda(miniconda) is pretty good stuff but I recommend againts it - if you dont want to look at a terminal spinner for hours while installing something (and have time to contemplate the absurd) you should use [mamba](https://mamba.readthedocs.io/en/latest/installation/mamba-installation.html).
**Mamba** is the same as conda but better – regarding what you care about – faster install times and has venom to deal with the pesky rodents. 

Now when you want to use a docker image that runs some job/task/script. 
Using conda could be a bit of a trouble – this dude explains it - https://kevalnagda.github.io/conda-docker-tutorial. But why not use `pip3 install -r requirements.txt`?!? And be done with our lives.. Because I dont like pip ok?!
And pip tends to do some weird shit especially when you need to install pytorch with cuda support or you already have installed pytorch and need to install transformers and pip decides to remove and reinstall the same version and fucking up stuff in the process?!?! K .
Back to the docker issue. So now that you know that you should use mamba(I know poetry is super cool but we are data sciencing here), regarding docker containers the best experience is using 
[micromamba](https://mamba.readthedocs.io/en/latest/installation/micromamba-installation.html) 

**WUT ONE MORE SNEK**,

yes, micromamba is even lighter version of mamba which comes even without pesky python in the “base” environment and no channels preconfigured (essentially no .condarc/.mambarc). What this means is running 

```sh
micromamba install -n base numpy
```

Will throw error complaining that that no channel was specified but 
```sh
micromamba install -n base numpy -c conda-forge
```
Is OK. 

When you want to run something within the image with **activated base environment** the easiest approach is to just use the micromamba base image like so:
```Dockerfile
FROM mambaorg/micromamba:latest

# just add link to docs
#ARG MAMBA_DOCKERFILE_ACTIVATE=1

# you can either add channels here and save the options to .condarc or use env.yml
# RUN micromamba config append channels conda-forge --env 

WORKDIR /app
COPY --chown=$MAMBA_USER:$MAMBA_USER . .

# -n name of environment should be 'base' even if the env_name in the yml file is env15
# you should specify channels in env.yml
RUN micromamba install -y -n base -f env.yml

CMD ["python", "do_stuff.py"]
```
The env.yml looks like this:
```yml
# the name does not matter we force it to be base in the build step
name: anyname
channels:
  - pytorch
  - nvidia
  - conda-forge
dependencies:
  - python=3.10
  - pytorch=2.1
  - pytorch-cuda=12.1
  - numpy
```
And do_stuff.py

```python
import torch

cuda_available = torch.cuda.is_available()
device = torch.device("cuda") if cuda_available else torch.device("cpu")

if cuda_available:
    print('CUDA available')
    print(torch.cuda.mem_get_info())
    arr = torch.rand((3000,3000), device=device)
else:
    print('CUDA not available')
    arr = torch.rand((3000,3000), device=device)

```

Next just build and run the container. 
```sh
docker build -t nocuda -f Dockerfile
docker run -rm nocuda
```
And do_stuff.py will run, my "do_stuff.py" just prints if cuda is available for this container the answer is False. 

Now if you want to do something more complicated or need functionallity from other base images, for example you want to run the fanciest LLMs in a container with GPU and brag about it on meetups or even better put a "Generative AI expert" on your Linkedin. The trick is to use the micro snake again but with the addition of nvidia image with needed drivers. If running on VM the host machine needs to have same NVIDIA drivers and NVIDIA Container Toolkit. 

Now following the great instruction on [micromamba docs](https://micromamba-docker.readthedocs.io/en/latest/advanced_usage.html#adding-micromamba-to-an-existing-docker-image) we arive at:

```Dockerfile
FROM mambaorg/micromamba:latest as micromamba

# Use NVIDIA drivers image as the base image
FROM nvcr.io/nvidia/driver:535.54.03-ubuntu22.04

RUN apt-get update -y

USER root

# if your image defaults to a non-root user, then you may want to make the
# next 3 ARG commands match the values in your image. You can get the values
# by running: docker run --rm -it my/image id -a
ARG MAMBA_USER=mambauser
ARG MAMBA_USER_ID=57439
ARG MAMBA_USER_GID=57439
ENV MAMBA_USER=$MAMBA_USER
ENV MAMBA_ROOT_PREFIX="/opt/conda"
ENV MAMBA_EXE="/bin/micromamba"

COPY --from=micromamba "$MAMBA_EXE" "$MAMBA_EXE"
COPY --from=micromamba /usr/local/bin/_activate_current_env.sh /usr/local/bin/_activate_current_env.sh
COPY --from=micromamba /usr/local/bin/_dockerfile_shell.sh /usr/local/bin/_dockerfile_shell.sh
COPY --from=micromamba /usr/local/bin/_entrypoint.sh /usr/local/bin/_entrypoint.sh
COPY --from=micromamba /usr/local/bin/_dockerfile_initialize_user_accounts.sh /usr/local/bin/_dockerfile_initialize_user_accounts.sh
COPY --from=micromamba /usr/local/bin/_dockerfile_setup_root_prefix.sh /usr/local/bin/_dockerfile_setup_root_prefix.sh

RUN /usr/local/bin/_dockerfile_initialize_user_accounts.sh && \
    /usr/local/bin/_dockerfile_setup_root_prefix.sh

USER $MAMBA_USER

SHELL ["/usr/local/bin/_dockerfile_shell.sh"]

ENTRYPOINT ["/usr/local/bin/_entrypoint.sh"]

WORKDIR /app
COPY --chown=$MAMBA_USER:$MAMBA_USER . .

RUN micromamba install -y -n base -f env.yml && \
    micromamba clean --all --yes

CMD ["python", "do_stuff.py"]
```

Now after building and running the container we should get "Cuda available", and thus we can use the GPU. Now you just need to replace this do_stuff with your do_stuff.

If you want to use this docker with `exec` command then you will need to activate the environment. But the canonical use case is to run it as a service in this case this means only **one environment** per docker image. But then why spent so much effort using mambas env ? Convenience mostly and getting the right packages and version, even better to use conda-lock files. I have always been using conda/mamba for envs and packages and my experience has been positive espeacially when something compiled is needed as torch-cuda version. 

