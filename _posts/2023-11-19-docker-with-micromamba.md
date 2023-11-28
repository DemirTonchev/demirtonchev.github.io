---
layout: post
title: "Using docker and mamba without tears"
categories: MLeng
tags: mamba docker
author:
- Me
---


If you are here, you probably know what [conda](https://docs.conda.io/en/latest/) is (open-source package manager and environment manager). This means that you can use it to _create_ virtual environments and _install_ packages within, also conda will take care of interdependencies of the packages and you won't get some nasty error. If a package is not available on conda channels you can always resolve to pip to install it.

### mamba
So conda(miniconda) is pretty good stuff but I recommend againts it - if you dont want to look at a terminal spinner for hours while installing something (and have time to contemplate the absurd) you should use [mamba](https://mamba.readthedocs.io/en/latest/installation/mamba-installation.html).
**Mamba** is the same as conda but better – regarding what you care about – faster install times and has venom to deal with the pesky rodents. Mamba supports most of the conda commands, so if you know conda you know mamba. If you dont know conda look at the documentation (but let's be honest no one reads documentaion and you will look at stackoverflow or ask chatGPT).

But you are here because you want to use a docker image that runs some script/program. 
Using conda could be a bit of a trouble – this dude explains some of the pain [https://kevalnagda.github.io/conda-docker-tutorial](https://kevalnagda.github.io/conda-docker-tutorial). But why not use `pip3 install -r requirements.txt`?!? And be done with our lives.. Because I dont like pip ok?!
And pip tends to do some weird stuff especially when you need to install pytorch with cuda support or you already have installed pytorch and need to install transformers and pip decides to remove and reinstall the same version of pytorch fucking up stuff in the process?!?! **K**.
Back to the docker issue. So now that you know that you should use mamba in general(I know poetry is super cool but we are data sciencing here), regarding docker containers the best experience is using 
[micromamba](https://mamba.readthedocs.io/en/latest/installation/micromamba-installation.html) 

**WUT ONE MORE SNEK**,

yes, micromamba is even lighter version of mamba which comes even without python in the “base” environment and no channels preconfigured (essentially no .condarc/.mambarc). What this means is running 

```sh
micromamba install -n base numpy
```

will throw error complaining that that no channel was specified but 
```sh
micromamba install -n base numpy -c conda-forge
```
is OK. 

When you want to run something within the image with **activated base environment** the easiest approach is to just use the micromamba base image like so:
```Dockerfile
FROM mambaorg/micromamba:latest

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
    print(torch.cuda.mem_get_info())
else:
    print('CUDA not available')
    arr = torch.rand((3000,3000), device=device)

```

next just build and run the container. 
```sh
docker build -t nocuda -f Dockerfile
docker run -rm nocuda
```
and do_stuff.py will run, my "do_stuff.py" just prints if CUDA is available for this container the answer is False. 

Now if you want to do something more complicated or need functionallity from other base images, for example you want to run the fanciest LLMs in a container with GPU and brag about it on meetups or even better put a "Generative AI expert" on your Linkedin. The trick is to use the micro snake again but with the addition of nvidia image with needed drivers. If running on VM the host machine needs to have same NVIDIA drivers and NVIDIA Container Toolkit. 

Now following the great instruction on [micromamba docs](https://micromamba-docker.readthedocs.io/en/latest/advanced_usage.html#adding-micromamba-to-an-existing-docker-image) we arive at:

```Dockerfile
FROM mambaorg/micromamba:latest as micromamba

# Use NVIDIA drivers image as the base image
# careful about the driver version
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
The only difference is at runtime

```sh
docker build -t yescuda -f Dockerfile
docker run --gpus all -t yescuda
```
After building and running the container we should get "Cuda available", and thus we can use the GPU. Now you just need to replace this do_stuff with your do_stuff.

If you want to use this docker with `exec` command then you will need to activate the environment. But the canonical use case is to run it as a service that does something specific. In this case this means only **one (base) environment** per docker image. But then why spent so much effort using mambas envs if we can use only the base one? Convenience and speed mostly and getting the right packages and version, if you want to be really punctual you can use conda-lock files. Also your development env.yml(you don't install everythign in "base" right? RIGHT?) is directly usable for the docker, meaning you would get the same results as per dev/experimentation. 

And that's it now you can use micromamba with docker with GPU. No tears as promised!


<sub> PS.
If you wonder that to do next check this envelope problem for fun - [https://en.wikipedia.org/wiki/Two_envelopes_problem](https://en.wikipedia.org/wiki/Two_envelopes_problem) </sub>


