# Docker 服务器配置

## 安装Docker

* 下载对应显卡和系统的Nvidia驱动并进行安装。网上教程较多，不予赘述，本文采用的是最新驱动。

* 安装Docker CE 19.03与NVIDIA Docker，该版本Docker已支持GPU，无需再安装Nvidia Docker 2。教程可参考清华镜像安装（比较快）以及参考官网和Nvidia Docker以及该教程。

```
# Add the package repositories
$ distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
$ curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
$ curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
 
$ sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
$ sudo systemctl restart docker

```

* 注意配置/etc/docker/daemon.json文件夹 否则创建容器时会出错
![daemon_json](./daemon_json.png "daemon_json")

* 创建Docker用户组并配置用户Docker权限

```
groupadd docker
gpasswd -a $USER docker

```

* 拉取nvidia/cuda:10.1-cudnn7-devel-ubuntu16.04镜像，其他支持cuda的镜像可以在这里找到
```
## 注意尽量跟本机的cuda版本匹配
docker pull nvidia/cuda:10.1-cudnn7-devel-ubuntu16.04

```

* CUDA的环境变量配置(.bashrc文件)
```
export PATH="/usr/local/cuda-10.1/bin:$PATH"
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/local/cuda/lib64"
export PATH="/usr/local/cuda/bin:$PATH"

```

* 赵老师服务器ip信息  
![zhaodi_server](zhaodi_server.png "zhaodi_server")

## 基于Dockerfile服务器配置

#### 编写需要的DockerFile (生信服务器为背景)

```Dockerfile_bio
FROM nvidia/cuda:10.1-cudnn7-devel-ubuntu16.04
MAINTAINER CSuperlei<csuperlei@163.com>

# ADD sources.list /etc/apt/sources.list

RUN apt upgrade -y && apt update -y && \
    mkdir -p /root/.cpan && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* /root/.cpan/*

RUN apt install -y wget curl net-tools iputils-ping locales  \
    unzip bzip2 apt-utils \
    tmux screen \
    git htop xclip cmake sudo tree jq \
    build-essential gfortran automake bash-completion \
    libapparmor1 libedit2 libc6 \
    psmisc rrdtool libzmq3-dev libtool apt-transport-https \
    && locale-gen en_US.UTF-8 && \
    apt install -y software-properties-common && \
    add-apt-repository ppa:ubuntugis/ppa -y && \
    apt update -y && \
    apt install -y bioperl libdbi-perl \
    supervisor \
    gdebi-core \
    openssh-server \
    libjansson-dev \
    libcairo2-dev libxt-dev \
    libv8-3.14-dev libudunits2-dev libproj-dev gdal-bin proj-bin libgdal-dev libgeos-dev libclang-dev && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* /root/.cpan/*
    #cpan -i Try::Tiny && \

# bash && ctags && cscope && gtags
RUN apt install cscope libncurses5-dev -y && \
    cd /tmp && \
    curl https://ftp.gnu.org/gnu/bash/bash-5.0.tar.gz -o bash-5.0.tar.gz && \
    tar xzf bash-5.0.tar.gz && cd bash-5.0 && ./configure && make && make install && \
    cd /tmp && \
    git clone --depth 1 https://github.com/universal-ctags/ctags.git && \
    cd ctags && ./autogen.sh && ./configure && make && make install && \
    cd /tmp && \
    curl http://ftp.vim.org/ftp/gnu/global/global-6.6.3.tar.gz -o global.tar.gz && \
    tar xzf global.tar.gz && cd global-6.6.3 && ./configure --with-sqlite3 && make && make install && \
    cd /tmp && \
    curl https://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.16.tar.gz -o libiconv.tar.gz && \
    tar xzf libiconv.tar.gz && cd libiconv-1.16 && ./configure && make && make install && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* /root/.cpan/*

# R language
RUN add-apt-repository 'deb https://cloud.r-project.org/bin/linux/ubuntu xenial-cran35/' && \
    apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 51716619E084DAB9 && \
    apt upgrade -y && apt update -y && \
    apt install -y r-base-dev r-base r-base-core && \
    apt install openjdk-8-jdk xvfb libswt-gtk-4-java -y && \
    R CMD javareconf && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* /root/.cpan/*

# rstudio
RUN cd /tmp && \
    curl https://download2.rstudio.org/server/trusty/amd64/rstudio-server-1.2.5033-amd64.deb -o rstudio.deb && \
    gdebi -n rstudio.deb && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* /root/.cpan/*

# language
RUN apt update && \
    apt install -y language-pack-zh-hans && locale-gen zh_CN.UTF-8 && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* /root/.cpan/*

# conda
ADD .condarc /root
ENV PATH=/opt/miniconda3/bin:$PATH
RUN cd /tmp && \
    rm -f /bin/bash && ln -s /usr/local/bin/bash /bin/bash && \
    curl https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-latest-Linux-x86_64.sh -o miniconda3.sh && \
    bash miniconda3.sh -b -p /opt/miniconda3 && \
    conda update -n base -c defaults conda pip && \
    conda clean -a -y && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* /root/.cpan/*

# conda install
RUN conda install -n base -c conda-forge time libxml2 libxslt libssh2 krb5 ripgrep lazygit yarn nodejs=12.16 jupyterlab=2.0.1 && \
    /opt/miniconda3/bin/pip install --no-cache-dir pynvim neovim-remote flake8 pygments ranger-fm msgpack-python jedi==0.15.2 && \
    /opt/miniconda3/bin/pip install --no-cache-dir python-language-server && \
    conda clean -a -y && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* /root/.cpan/*

# RUN /opt/miniconda3/bin/pip install --no-cache-dir pandas scikit-learn numpy matplotlib scipy seaborn ggplot plotly xgboost && \
#     conda clean -a -y && \
#     apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* /root/.cpan/*

# vim
RUN cd /tmp && \
    git clone --depth 1 https://github.com/vim/vim.git && \
    cd vim && \
    export LDFLAGS='-L/opt/miniconda3/lib -Wl,-rpath,/opt/miniconda3/lib' && \
    ./configure --with-features=huge \
      --enable-multibyte \
      --enable-python3interp=yes \
      --with-python3-config-dir=/opt/miniconda3/lib/python3.7/config-3.7m-x86_x64-linux-gnu \
      --prefix=/usr/local && \
    make -j16 && make install && \
    rm -rf /tmp/* /var/tmp/* /root/.cpan/*

# nvim
RUN cd /usr/local && \
    curl -L https://github.com/neovim/neovim/releases/download/v0.4.3/nvim-linux64.tar.gz -o nvim-linux64.tar.gz && \
    tar xzf nvim-linux64.tar.gz && \
    rm nvim-linux64.tar.gz && \
    ln -s /usr/local/nvim-linux64/bin/nvim /usr/local/bin/nvim

# coder server
RUN cd /tmp && \
    curl -L https://github.com/cdr/code-server/releases/download/2.1698/code-server2.1698-vsc1.41.1-linux-x86_64.tar.gz -o code-server.tar.gz && \
    tar xzf code-server.tar.gz && \
    mv code-server2.1698-vsc1.41.1-linux-x86_64 /opt/code-server && \
    rm -rf /tmp/*.*

# configuration
RUN mkdir -p /etc/rstudio /opt/config /opt/log /opt/rc && chmod -R 755 /opt/config /opt/log
COPY .bashrc .inputrc /opt/rc/
## users ports and dirs and configs
RUN echo "export.UTF-8" >> /etc/profilesource /etc/profile
RUN echo "export LC_ALL='C.UTF-8'" >> /etc/profile
ENV LANG C.UTF-8
ENV WKUID=1000
ENV WKUSER=datasci
ENV PASSWD=datasci
ENV COUNTRY=CN
ENV PROVINCE=ZJ
ENV CITY=HZ
ENV ORGANIZE=SELF
ENV WEB=leatchina.data.sci
ENV IP=0.0.0.0
ENV CHOWN=1
ENTRYPOINT ["bash", "/opt/config/entrypoint.sh"]
EXPOSE 8888 8787 8686 8585
## config file
COPY rserver.conf /etc/rstudio/

COPY jupyter_lab_config.py supervisord.conf passwd.py entrypoint.sh /opt/config/

```

#### 编译镜像

* 在有Dockerfile的文件目录中运行 docker build 命令
```
    docker build -t 仓库名:标签 .
    # 例如
    docker build -t ubuntu/test:v1.0
```

#### 从镜像创建容器
```
    docker run --name 容器名 -v 宿主机目录:容器目录 -it 镜像名
    # 例如
    docker run --name buct-cailei --gpus all -v /home/cailei:/home/cailei -itd buct/cailei-anaconda:v0.1-3 /bin/bash  (--rm 是退出容器时将当前容器删除)
    
```

#### 进入容器

```
    docker exec -it buct-cailei /bin/bash
    ## 查看gpu是否可以使用
    nividia-smi
    nvcc --version
```

#### 退出容器

'''

    exit

    ctrl + D
'''

#### 查看全部容器(包括停止的)
```

    docker ps -a 

```


#### 查看镜像信息

```
    docker inspect 镜像名

```


#### 重启容器
```

    docker start buct-cailei

```


#### Conda虚拟环境

* 查看当前Conda虚拟环境

```

    conda env list

```

* 创建虚拟环境

```

    conda create -n csuperlei_bio python=3.6

```

* 进入虚拟环境(激活虚拟环境)

```

    source activate csuperlei_bio

```


* 安装必要软件

```
    ## 安装pytorch -c 是通道的意思
    conda install pytorch torchvision -c pytorch
    ## 安装tensorflow
    conda install tensorflow-gpu
```


* 退出虚拟环境

```

    conda deactivate

```


* 删除虚拟环境

```

    conda remove -n csuperlei_bio --all

```


#### Docker Pytorch Tensorflow

```
FROM nvidia/cuda:10.1-cudnn7-devel-ubuntu16.04
MAINTAINER CSuperlei<csuperlei@163.com>

ADD sources.list /etc/apt/sources.list

RUN apt upgrade -y && apt update -y && \
    mkdir -p /root/.cpan && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* 

RUN apt install -y wget curl net-tools iputils-ping locales  \
    zip unzip bzip2 apt-utils zlib1g zlib1g-dev \
    tmux screen \
    gcc g++ \
    vim git htop xclip cmake sudo tree jq \
    build-essential gfortran automake bash-completion \
    libapparmor1 libedit2 libc6 \
    psmisc rrdtool libzmq3-dev libtool apt-transport-https \
    && locale-gen en_US.UTF-8 && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* 

# bash && ctags && cscope && gtags
RUN apt install cscope libncurses5-dev -y && \
    cd /tmp && \
    curl https://ftp.gnu.org/gnu/bash/bash-5.0.tar.gz -o bash-5.0.tar.gz && \
    tar xzf bash-5.0.tar.gz && cd bash-5.0 && ./configure && make && make install && \
    cd /tmp && \
#   git clone --depth 1 https://github.com/universal-ctags/ctags.git && \
#   cd ctags && ./autogen.sh && ./configure && make && make install && \
#   cd /tmp && \
#   curl http://ftp.vim.org/ftp/gnu/global/global-6.6.3.tar.gz -o global.tar.gz && \
#   tar xzf global.tar.gz && cd global-6.6.3 && ./configure --with-sqlite3 && make && make install && \
#   cd /tmp && \
#   curl https://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.16.tar.gz -o libiconv.tar.gz && \
#   tar xzf libiconv.tar.gz && cd libiconv-1.16 && ./configure && make && make install && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* 

# R language
RUN add-apt-repository 'deb https://cloud.r-project.org/bin/linux/ubuntu xenial-cran35/' && \
    apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 51716619E084DAB9 && \
    apt upgrade -y && apt update -y && \
    apt install -y r-base-dev r-base r-base-core && \
    apt install openjdk-8-jdk xvfb libswt-gtk-4-java -y && \
    R CMD javareconf && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* 

# rstudio
RUN cd /tmp && \
    curl https://download2.rstudio.org/server/trusty/amd64/rstudio-server-1.2.5033-amd64.deb -o rstudio.deb && \
    gdebi -n rstudio.deb && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* 

# language
RUN apt update && \
    apt install -y language-pack-zh-hans && locale-gen zh_CN.UTF-8 && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* 

# conda
ADD .condarc /root
ENV PATH=/opt/miniconda3/bin:$PATH
RUN cd /tmp && \
    rm -f /bin/bash && ln -s /usr/local/bin/bash /bin/bash && \
    curl https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-latest-Linux-x86_64.sh -o miniconda3.sh && \
    bash miniconda3.sh -b -p /opt/miniconda3 && \
    conda update -n base -c defaults conda pip && \
    conda clean -a -y && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* 

# conda install
RUN conda install -n base -c conda-forge time libxml2 libxslt libssh2 krb5 ripgrep lazygit yarn nodejs=12.16 jupyterlab=2.0.1 && \
    /opt/miniconda3/bin/pip install --no-cache-dir -i http://pypi.douban.com/simple --trusted-host pypi.douban.com pynvim neovim-remote flake8 pygments ranger-fm msgpack-python jedi==0.15.2 && \
    /opt/miniconda3/bin/pip install --no-cache-dir -i http://pypi.douban.com/simple --trusted-host pypi.douban.com python-language-server && \
    conda clean -a -y && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* 

# machine learning
RUN /opt/miniconda3/bin/pip install --no-cache-dir -i http://pypi.douban.com/simple --trusted-host pypi.douban.com pandas scikit-learn numpy matplotlib scipy seaborn ggplot plotly xgboost && \
    conda clean -a -y && \
    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* 

## vim
RUN cd /tmp && \
    git clone --depth 1 https://github.com/vim/vim.git && \
    cd vim && \
    export LDFLAGS='-L/opt/miniconda3/lib -Wl,-rpath,/opt/miniconda3/lib' && \
    ./configure --with-features=huge \
      --enable-multibyte \
      --enable-python3interp=yes \
      --with-python3-config-dir=/opt/miniconda3/lib/python3.7/config-3.7m-x86_x64-linux-gnu \
      --prefix=/usr/local && \
    make -j16 && make install && \
    rm -rf /tmp/* /var/tmp/* /root/.cpan/*

# conda env deeplearning
#RUN conda create -n lei_env python=3.6 && \
#    source activate lei_env && \
#    conda install pytorch torchvision -c pytorch && \
#    conda install tensorflow-gpu keras && \
#    conda clean -a -y && \
#    conda deactivate && \
#    apt autoremove -y && apt clean -y && apt purge -y && rm -rf /tmp/* /var/tmp/* 

```





