---
layout: post
title: "Using docker and mamba without tears"
categories: MLeng
tags: mamba docker
author:
- Me
---

If you are here, you probably know what conda is (open-source package management system and environment management system). This means that you can use it to create_ virtual environments and _install_ packages within, also conda will take care of interdependencies of the packages. If a package is not available on conda channels you can always resolve to pip to install it.

So conda(miniconda3) is good but I recommend againts it - if you dont want to look at a terminal spinner for hours(If you ever experienced a painfully slow install with conda you will know.) you should use [mamba](https://mamba.readthedocs.io/en/latest/installation/mamba-installation.html).
**Mamba** is the same as conda but better – what you care about – faster install times. Now when you want to use a docker image that runs some job/task/script.
Using conda could be a bit of a trouble – this dude explains it - https://kevalnagda.github.io/conda-docker-tutorial. But why not use `pip3 install -r requirements.txt`?!? And be done with our lives.. Because I dont like pip ok?!
And pip tends to do some weird shit especially when you need to install pytorch with cuda support or you already have installed pytorch and need to install transformers and pip decides to remove and reinstall the same version and fucking up stuff in the process?!?! K .
Back to the docker issue. So now that you know that you should use mamba(I know poetry is super cool but we are data sciencing here), regarding docker containers the best experience is using 
[micromamba](https://mamba.readthedocs.io/en/latest/installation/micromamba-installation.html) 

**WUT ONE MORE SNEK**,

yes, micromamba is even lighter version of mamba which comes even without pesky python in the “base” environment and no channels preconfigured (essentially no .condarc/.mambarc). What this means is running 

```
micromamba install -n base numpy
```

Will throw error complaining that that no channel was specified but 
```
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

WORKDIR /embed_pipe
COPY --chown=$MAMBA_USER:$MAMBA_USER . .

# -n name of environment should be 'base' even if the env_name in the yml file is env15
# you should specify channels in env.yml
RUN micromamba install -y -n base -f env.yml

CMD ["python", "do_stuff.py"]
```

Next just build and run the container. 
```sh
docker build .......asdsdasd 
docker run -rm ....
```
And do_stuff.py will run, my "do_stuff.py" just prints if cuda is available for this container the answer is False. 

Now if you want to do something more complicated or need functionallity from other base images, for example you want to run the fanciest LLMs on a container with GPU.

