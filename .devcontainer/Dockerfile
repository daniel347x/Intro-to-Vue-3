############################################################
# v2.1.0.0 #v2.1.0.0 #docker-active
############################################################

FROM nvidia/cuda:11.1-devel-ubuntu20.04

# Force full rebuild of the Docker image by incrementing the version number in the line below, whenever necessary
RUN export DUMMY="version-201" && touch ${DUMMY} && rm ${DUMMY}

###
# See https://github.com/NVIDIA/nvidia-docker/issues/1632#issuecomment-1112667716
###
RUN rm /etc/apt/sources.list.d/cuda.list
RUN rm /etc/apt/sources.list.d/nvidia-ml.list

RUN apt-get update && apt-get -y upgrade

ENV DEBIAN_FRONTEND noninteractive

###
# Installations required to download/unzip other installations...
# and related stuff in the category of downloading/zipping.
# ...Except for installations in this category requiring 'pip' to install,
# because 'pip' hasn't been installed yet.
###
RUN apt-get install -y wget ca-certificates git sudo p7zip-full unzip bzip2 zip

###
# Python
###
RUN ln -fs /usr/share/zoneinfo/America/New_York /etc/localtime && apt-get install -y tzdata software-properties-common \
    && add-apt-repository ppa:deadsnakes/ppa \
    && apt-get update \
    && apt-get install -y python3.9 python3.9-dev make automake gcc g++ \
    && rm /usr/bin/python3 \
    && ln -sv /usr/bin/python3.9 /usr/bin/python3 \
    && ln -sv /usr/bin/python3 /usr/bin/python \
    && apt-get install -y python3.9-distutils python3-apt \
    && wget https://bootstrap.pypa.io/get-pip.py && python get-pip.py && rm get-pip.py \
    && pip install --upgrade pip
#
# Would have installed 'py7zr' with the other 'zip'-related installations above,
# but pip wasn't installed yet, so install it now.
# Also install the time-zone component for Python, because installing this first thing
# after installing Python seems like the right place to do it.
#
RUN pip install --upgrade py7zr pytz \
    ###
    # PyTorch and various other related core installations
    ###
    && pip install --upgrade tqdm \
    && pip install --upgrade torch torchvision torchaudio tensorboard cython torchsummary --extra-index-url https://download.pytorch.org/whl/cu113 \
    && pip install --upgrade dominate requests dataclasses dataclasses-json scipy opencv-python debugpy pandas tables scikit-learn statsmodels \
    && pip install --upgrade visdom fire pyyaml tensorboardX wandb pytorch_memlab pytorch-model-summary ray \
    # Profilers
    && pip install --upgrade psutil \
    && pip install --upgrade memory_profiler pyinstrument yappi \
    && pip install -U scalene \
    && pip install timm \
    && pip install einops

###
# A special SDK for wandb is required to overcome file-size limits for artifact uploads via an Azure back-end Enterprise installation
# Also, the version of wandb in 20.04 is not the latest and we want it for various improvements made in conjunction with discussion with W&B.
###
RUN pip install --upgrade wandb
RUN pip install --upgrade wandb[azure]

###
# MySQL: Because when you need it, you'll probably need it urgently, be up super late, need it right then,
# and not want to bother to figure out how to get it installed and accessible remotely over the VPN.
# Whoala! Here you go
###
# To run MySQL in the container for the first time (following times should be fine too, I think, or else remove the user creation part):
# Change password when actually running this command for proper security:
# RUN sudo service mysql start && sudo mysql --user=root mysql -e "CREATE USER 'deepink'@'%' IDENTIFIED BY 'deepink'; grant all privileges on *.* to 'deepink'@'%' WITH GRANT OPTION; GRANT SUPER ON *.* TO 'deepink'@'%'; flush privileges;"
###
# To access MySQL, use any remote MySQL client and access the host at port 3306 directly, assuming the container is run with its network shared with the host's
###
RUN apt-get update && apt-get install -y libsm6 libxext6 ffmpeg libxrender-dev mysql-server \
    && sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/mysql.conf.d/mysqld.cnf \
    && pip install --upgrade mysql-connector-python

###
# Redis - it's just too convenient at those times you could take advantage of it not to have it ready and installed.
# See https://redis.io/topics/quickstart
###
# Redis Server
RUN wget http://download.redis.io/redis-stable.tar.gz \
    && tar xvzf redis-stable.tar.gz \
    && cd redis-stable && make \
    && cp src/redis-server /usr/local/bin/ && cp src/redis-cli /usr/local/bin/ && mkdir /etc/redis \
    && cp utils/redis_init_script /etc/init.d/redis_6379 \
    && cp redis.conf /etc/redis/6379.conf \
    && sed -i 's/daemonize no/daemonize yes/g' /etc/redis/6379.conf \
    && sed -i 's|logfile ""|logfile /data/redis/log/redis_6379.log|g' /etc/redis/6379.conf \
    && sed -i 's|dir ./|dir /data/redis/data/6379|g' /etc/redis/6379.conf \
    && update-rc.d redis_6379 defaults \
    # Redis CLI Client
    && pip install --upgrade redis

###
# SQLite
###
RUN sudo apt-get install -y sqlite3 \
    && pip install --upgrade sqlite-web

###
# create non-root user
###
ENV USER=deepink
# A safe, custom UID
ENV UID=2000
RUN sudo groupadd deepink \
    && useradd -m --no-log-init --system --uid ${UID} ${USER} -g deepink -G sudo \
    && echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
USER ${USER}
WORKDIR /home/${USER}

###
# Switch default from /bin/sh to /bin/bash
# ...required by NVM/Node automated installation, below,
# and just generally what most of us want and expect.
###
SHELL ["/bin/bash", "--login", "-c"]

###
# NVM / Node
###
RUN sudo apt-get install -y curl \
    && curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash \
    && export NVM_DIR="$HOME/.nvm" && \. "$NVM_DIR/nvm.sh" && \. "$NVM_DIR/bash_completion" && nvm ls-remote \
    && export NVM_DIR="$HOME/.nvm" && \. "$NVM_DIR/nvm.sh" && \. "$NVM_DIR/bash_completion" && nvm install v16

###
# Conda
# Note: Because we are ourselves an image, there's much less need to install virtualized environments.
# But Conda works fine on top of 'pip' as long as you simply stick with the default Conda environment -
# it will use your existing and following pip installations just fine.
# We install Conda because some installations are easier using Conda than pip.
###
RUN cd ~ && wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ./miniconda.sh && sudo chmod ugo+x ./miniconda.sh && ls -l && bash ./miniconda.sh -b -p $HOME/miniconda

###
# Conda installations here (generally, those with no pip installations)
###

# HiddenLayer + graphviz prerequisite
RUN sudo apt-get install -y graphviz \
    && pip install graphviz hiddenlayer \
    ###
    # Ruby
    # ...for those components that require it, and some do
    ###
    && sudo apt-get install -y rubygems \
    ###
    # SSHD
    ###
    && sudo apt-get install -y openssh-server \
    # ###
    # # More cool stuff
    # ###
    # RUN curl -LO https://github.com/BurntSushi/ripgrep/releases/download/12.1.1/ripgrep_12.1.1_amd64.deb
    # RUN sudo dpkg -i ripgrep_12.1.1_amd64.deb
    && sudo apt-get install -y ripgrep tree cpio \
    && echo "export RIPGREP_CONFIG_PATH=/home/${USER}/.ripgrep" >> ${HOME}/.bashrc

###
# Jupyter
# Note: To run the notebook server: NAVIGATE TO HOME DIRECTORY: jupyter notebook --ip 0.0.0.0 --port 8888 --no-browser
# To access the notebook: http://<ip>:8888 - and enter token to be found in the server console after running the notebook server
###
# # https://stackoverflow.com/a/47800763/368896 - but uninstall instead; and this MUST occur in its own RUN line or else the error that follows is not resolved
# RUN sudo pip uninstall -y pyzmq
# I do not know why the following must run on its own line to prevent error "ImportError: cannot import name 'constants' from partially initialized module 'zmq.backend.cython' (most likely due to a circular import) (/usr/lib/python3/dist-packages/zmq/backend/cython/__init__.py)"
RUN pip install --upgrade jupyterlab
RUN sudo apt-get install -y jupyter-core jupyter-notebook \
    && cd ~ && echo "export PATH=${PATH}:/home/${USER}/.local/bin" >> ./.bashrc \
    && cd ~ && jupyter notebook --generate-config \
    # Allow remote access without hassle
    && cd ~ && sed -i "s/# c.NotebookApp.allow_origin = ''/c.NotebookApp.allow_origin = '*'/g" ~/.jupyter/jupyter_notebook_config.py \
    && cd ~ && sed -i 's/# c.NotebookApp.allow_remote_access = False/c.NotebookApp.allow_remote_access = True/g' ~/.jupyter/jupyter_notebook_config.py \
    && cd ~ && sed -i 's/# c.NotebookApp.allow_root = False/c.NotebookApp.allow_root = True/g' ~/.jupyter/jupyter_notebook_config.py \
    && cd ~ && sed -i 's/# c.NotebookApp.disable_check_xsrf = False/c.NotebookApp.disable_check_xsrf = True/g' ~/.jupyter/jupyter_notebook_config.py

###
# vim and Ranger
###
RUN sudo apt-get install -y vim \
    && sudo apt-get install -y ranger \
    ###
    # Other installations
    ###
    && pip install --upgrade imgaug tabulate validators \
    && sudo gem install mdless \
    && export NVM_DIR="$HOME/.nvm" && \. "$NVM_DIR/nvm.sh" && npm install -g vtop \
    && git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf && ~/.fzf/install --all
#
# Yet other miscellaneous installations
RUN pip install jsonlines icecream h5py xmltodict rdp json2xml progressbar \
    && pip install -U ipywidgets

##########################################
# THE FOLLOWING BLOCK CAN BE IGNORED UNLESS YOU WANT TO ATTEMPT MANUAL QUANTIZATION, NOT USING ANY QUANTIZATION API
# ...Not fully cleaned up
##########################################
# Lower version required because UNSQUEEZE operator switched from the 'axes' argument being an attribute to being an input starting with 1.8.0 which breaks quantization using ONNX 1.8.0 because that version does not support any prior version of the UNSQUEEZE operator and when it saves an updated model it assumes the UNSQUEEZE operator has 2 inputs but the exported v7 version of the ONNX file has only one input (and one attribute)
# RUN pip install onnx==1.7.0
# REQUIREMENTS for onnx_ml_pb2 (onnx protobuf compiler required for ONNX quantization)
# install onnxruntime AFTER the above (?? - couldn't load quantized ONNX, maybe due to https://github.com/microsoft/onnxruntime/issues/5243#issuecomment-696804055)
# RUN pip install onnxruntime==1.7.2 # unavailable as of Monday, May 3, 2021 1:46 AM
# RUN pip install onnxruntime==1.7.0
# RUN sudo apt-get install -y protobuf-compiler
# I tried both the following but https://github.com/microsoft/onnxruntime/issues/5243#issuecomment-696804055 pointed me to the fact that apt-get installed v2.4.3. But note: 2.4.3 -> 2.6.2 did not solve the problem of starting a session with the fp16 model at node 0 with input name 865
# RUN pip install pybind11
# RUN sudo apt-get install python3-pybind11
# RUN pip install pybind11==2.6.2
# MUST GIT CLONE MANUALLY version 1.7.0 via git clone https://github.com/onnx/onnx.git followed by git checkout 0c070abb0c40fec649f81a73a75b0098662ec486 followed by wget https://github.com/Kitware/CMake/releases/download/v3.20.2/cmake-3.20.2-linux-x86_64.sh; bash cmake-3.20.2-linux-x86_64.sh - then add cmake to path via editing both .bashrc and .zshrc because sudo does not gain access to PATH when exported at command line -  PATH="/home/danissen/cmake-3.20.2-linux-x86_64/bin:$PATH" - but this still does not give SUDO access to PATH modifications - you must follow steps at https://superuser.com/a/927599/87699 namey sudo visudo then add the noted path to secure_path - followed by sudo python setup.py develop - nope - you must run the previous command in a sudo su shell after export PATH="..." (as before)

###
# A suite of useful installations
###
RUN pip install --upgrade docformatter \
    && pip install onnx \
    && pip install onnxruntime \
    && pip install onnxmltools \
    && pip install pyperclip \
    && pip install pytorch-lightning \
    && pip install rectpack \
    && pip install pydot \
    && pip install submitit \
    && pip install regex \
    && python -m pip install "dask[complete]" \
    && pip install bokeh \
    && pip install feather-format \
    && pip install -U jsonpickle \
    # Top2Vec requires gensim<4.0.0
    && pip install 'gensim<4.0.0' \
    && pip install -U spacy[cuda111,transformers,lookups] \
    && pip install transformers \
    && pip install sentencepiece \
    && pip install overrides \
    && pip install keybert \
    && pip install nltk \
    && pip install gitpython \
    && pip install hnswlib \
    && pip install sentence_transformers \
    && pip install wordcloud \
    && pip install umap-learn>=0.5.1 \
    && pip install hdbscan>=0.8.27 \
    && pip install wordcloud \
    && pip install timm

###
# APT installations
###
RUN    sudo apt install -y imagemagick \
    && sudo apt-get install -y pngquant \
    && sudo apt-get install -y iproute2

# Only if necessary - this is a hassle with conflicts
# RUN onnxruntime-gpu

###
# Ok, let's do it! Oh-my-zsh: https://github.com/ohmyzsh/ohmyzsh
# If you don't know about, or use, zsh (yes, instead of Bash)...
# you'll see it in action when you run the container! Enjoy.
###
# First, install zsh
RUN sudo apt-get install -y zsh
# OPTIONAL -> Switch default from /bin/sh to /usr/bin/zsh
# Can also use CMD, below; can also pass 'zsh' at the command line when executing the container.
# Having 'zsh' set up in all three ways makes it more robust against edge-case ways the console makes an appearance.
# For the occasional times when you really do need 'bash' as the default - sorry! There are pros and cons to everything.
SHELL ["/usr/bin/zsh", "--login", "-c"]
# Alternative to above - commented out
# RUN sudo sed -i 's/\/home\/${USER}:\/bin\/sh/\/home\/${USER}:\/usr\/bin\/zsh/g' /etc/passwd
# Second, install oh-my-zsh
RUN sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

###
# Handy change in the ZSH shell:
# Change prompt to make it clear we're in this Docker container, and not on the host's command line
###
RUN sed -i 's/➜/docker ➜/g' /home/${USER}/.oh-my-zsh/themes/robbyrussell.zsh-theme

###
# NVM / Node, this time in ZSh
###
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
RUN export NVM_DIR="$HOME/.nvm" && \. "$NVM_DIR/nvm.sh" && \. "$NVM_DIR/bash_completion" && nvm ls-remote
RUN export NVM_DIR="$HOME/.nvm" && \. "$NVM_DIR/nvm.sh" && \. "$NVM_DIR/bash_completion" && nvm install v16
RUN echo "export NVM_DIR=\"\$HOME/.nvm\" && \. \"\$NVM_DIR/nvm.sh\"" >> $HOME/.bashrc

# Make node and npm available without including the full path
# https://stackoverflow.com/a/5062718/368896
RUN    cd $HOME \
    && export NVM_DIR="$HOME/.nvm" \
    && \. "$NVM_DIR/nvm.sh" \
    && sudo ln -s $(which node) /usr/bin/node \
    && sudo ln -s $(which npm) /usr/bin/npm

###
# Setuptools must be updated from pre-installed version above
###
RUN pip install --upgrade setuptools

###
# This OS UTF-8 bullshit is required because the 'click' component,
# which is a deeply-nested dependency of one of the installed components above
# (I'm not sure which; nor am I sure that component is still being installed
# nor if it's necessary any more), is an old component that assumes Python defaults to ASCII encoding
# (even though Python 3 defaults to UTF-8, as all languages have for years now)
# and actually CHECKS the following envvars and raises an exception if they're not set to UTF-8.
# I'm not sure if the installation of 'locales' is required for that component
# and/or the following flags to somehow take effect,
# but I read a suggestion to include it (I think at the link below);
# I did include it; it works; and I haven't tested not including it yet.
# It doesn't seem to break anything or slow down anything so it just stays here.
# If anybody knows better why this is necessary and whether it can
# be removed - if it can be removed - please do so!
###
RUN sudo apt-get install -y locales
ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8
# https://serverfault.com/a/801162/108055
RUN echo "export LC_ALL=C.UTF-8" >> $HOME/.bashrc \
    && echo "export LANG=C.UTF-8" >> $HOME/.bashrc

###
# Useful bash shortcuts
###
RUN echo "export NVM_DIR=\"\$HOME/.nvm\" && \. \"\$NVM_DIR/nvm.sh\"" >> $HOME/.bashrc \
    && echo "alias top=vtop" >> $HOME/.bashrc

###
# Also populate ~/.zshrc with all of the same modifications as ~/.bashrc
###
RUN echo "export LC_ALL=C.UTF-8" >> $HOME/.zshrc \
    && echo "export LANG=C.UTF-8" >> $HOME/.zshrc \
    && cd ~ && echo "export PATH=${PATH}:/home/${USER}/.local/bin" >> ./.zshrc \
    && echo "export NVM_DIR=\"\$HOME/.nvm\" && \. \"\$NVM_DIR/nvm.sh\"" >> $HOME/.zshrc \
    && echo "alias top=vtop" >> $HOME/.zshrc
