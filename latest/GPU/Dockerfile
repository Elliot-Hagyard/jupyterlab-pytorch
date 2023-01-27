FROM nvcr.io/nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu20.04
SHELL ["/bin/bash", "-c"]
USER root

#Bringing in Jupyter stack elements
ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"
# Avoid issues with timezone
ENV TZ=US
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
# Install a few dependencies for iCommands, text editing, and monitoring instances
RUN apt update -y && \
    apt install -y lsb-release apt-transport-https curl gnupg2 libfuse2 gettext gcc less nodejs software-properties-common apt-utils glances htop nano wget


RUN wget -qO - https://packages.irods.org/irods-signing-key.asc | apt-key add - && \
    echo "deb [arch=amd64] https://packages.irods.org/apt/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/renci-irods.list && \
    apt update && \
    wget -q -c \
    http://security.ubuntu.com/ubuntu/pool/main/p/python-urllib3/python-urllib3_1.22-1ubuntu0.18.04.2_all.deb \
    http://security.ubuntu.com/ubuntu/pool/main/r/requests/python-requests_2.18.4-2ubuntu0.1_all.deb \
    http://security.ubuntu.com/ubuntu/pool/main/o/openssl1.0/libssl1.0.0_1.0.2n-1ubuntu5.10_amd64.deb && \
    apt install -y \
    ./python-urllib3_1.22-1ubuntu0.18.04.2_all.deb \
    ./python-requests_2.18.4-2ubuntu0.1_all.deb \
    ./libssl1.0.0_1.0.2n-1ubuntu5.10_amd64.deb && \
    rm -rf \
    ./python-urllib3_1.22-1ubuntu0.18.04.2_all.deb \
    ./python-requests_2.18.4-2ubuntu0.1_all.deb \
    ./libssl1.0.0_1.0.2n-1ubuntu5.10_amd64.deb && \
    apt install -y irods-icommands && \
    apt clean && \
    rm -rf /var/lib/apt/lists/*


RUN wget --quiet https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda.sh && \
    /bin/bash ~/miniconda.sh -b -p /opt/conda


ENV CONDA_DIR=/opt/conda \
    SHELL=bash \
    NB_USER="${NB_USER}" \
    NB_UID=${NB_UID} \
    NB_GID=${NB_GID} \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8

ENV PATH="${CONDA_DIR}/bin:${PATH}" \
    HOME="/home/${NB_USER}"

# Install Go
RUN wget -q -c https://dl.google.com/go/go1.17.6.linux-amd64.tar.gz -O - | sudo tar -xz -C /usr/local
ENV PATH=$PATH:/usr/local/go/bin 

# Install Rust
RUN apt update && \
    apt install -y cargo rustc

# Install CyberDuck CLI
RUN echo "deb https://s3.amazonaws.com/repo.deb.cyberduck.io stable main" > /etc/apt/sources.list.d/cyberduck.list && \
    apt-key adv --keyserver keyserver.ubuntu.com --recv-keys FE7097963FEFBE72 && \
    apt update && \
    apt install duck

# Install GitHub CLI
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/etc/apt/trusted.gpg.d/githubcli-archive-keyring.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/trusted.gpg.d/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" > /etc/apt/sources.list.d/github-cli.list && \
    apt update && \
    apt install -y gh


# Add sudo to jovyan user
RUN apt update && \
    apt install -y sudo && \
    apt clean && \
    rm -rf /var/lib/apt/lists/*
#
ARG LOCAL_USER=jovyan
ARG PRIV_CMDS='/bin/ch*,/bin/cat,/bin/gunzip,/bin/tar,/bin/mkdir,/bin/ps,/bin/mv,/bin/cp,/usr/bin/apt*,/usr/bin/pip*,/bin/yum'

RUN useradd -m -d /home/jovyan -u 1001 jovyan \
    && chown -R jovyan:jovyan /home/jovyan
ENV SHELL=bash

RUN usermod -aG sudo jovyan && \
    echo "$LOCAL_USER ALL=NOPASSWD: $PRIV_CMDS" >> /etc/sudoers

# purge older nodejs
RUN apt purge nodejs -y

RUN chown -R 1001:1001 /opt/conda

USER jovyan

WORKDIR /home/jovyan

RUN conda install -n base\
    pytorch torchvision torchaudio pytorch-cuda=11.7 -c pytorch -c nvidia

RUN conda update -n base conda \
    && conda install -c conda-forge mamba --yes \
    && mamba install -c conda-forge --yes nodejs jupyterlab ipykernel && \
    jupyter lab build

RUN conda init && \
    echo ". /opt/conda/etc/profile.d/conda.sh" >> /home/jovyan/.bash_profile && \
    echo "conda deactivate" >> /home/jovyan/.bash_profile && \
    echo "conda activate pytorch" >> /home/jovyan/.bash_profile 
RUN echo "source .bash_profile" >> /home/jovyan/.bashrc

RUN . /opt/conda/etc/profile.d/conda.sh && \
    conda activate pytorch && python -m ipykernel install --name "jovyan" --user
RUN source /home/jovyan/.bash_profile

# Install and configure jupyter lab.
COPY --chown=jovyan:jovyan jupyter_notebook_config.json /opt/conda/envs/pytorch/etc/jupyter/

# install iRODS pluginx
RUN pip install jupyterlab_irods==3.0.*

RUN conda install -c conda-forge pytorch-lightning opencv && conda install -c rapidsai-nightly mlflow -c conda-forge
# Extra copy of the conda environment in case we need it
COPY environment.yml /home/jovyan/environment.yml

# Resolves error for libcudart.so
RUN echo "export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda-11.3/targets/x86_64-linux/lib && source /etc/profile" >> /home/jovyan/.bashrc
EXPOSE 8888
COPY entry.sh /bin
RUN mkdir -p /home/jovyan/.irods

ENTRYPOINT ["/bin/bash", "/bin/entry.sh"]
