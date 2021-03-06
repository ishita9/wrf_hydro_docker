###################################
# Author: Joe Mills
# Author: Katelyn FitzGerald
# Date:   10.15.2019
###################################
#
# This Dockerfile compiles coupled WRF | WRF-Hydro and WPS from source for training purposes.

FROM ubuntu:xenial
MAINTAINER katelynw@ucar.edu

####################################
########## ROOT USER  ##############
####################################
USER root

####################################
##Additional linux and command-line tools
#Install add-apt-repository. This needs to be done starting Ubuntu 16.x
RUN apt-get update \
	&& apt-get install -yq --no-install-recommends \
	software-properties-common \
	&& apt-get clean \
	&& rm -rf /var/lib/apt/lists/*

####################################
##WRF, WRF-Hydro, and WPS dependencies and tools used for compilation
RUN apt-get update \
    && apt-get install -yq --no-install-recommends \
    vim \
    nano \
    wget \
    curl \
    file \
    bzip2 \
    ca-certificates \
    libhdf5-dev \
    gfortran \
    openmpi-bin \
    libopenmpi-dev \
    valgrind \
    m4 \
    make \ 
    libswitch-perl \
    git \
    csh \
    bc \
    libpng-dev \
    libjasper-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* 

## install netCDF-C
ENV NCDIR=/usr/local
ENV NFDIR=/usr/local
ENV H5DIR=/usr/lib/x86_64-linux-gnu/hdf5/serial
ENV HDF5_DIR=/usr/lib/x86_64-linux-gnu/hdf5/serial

RUN NETCDF_C_VERSION="4.4.1.1" \
    && wget ftp://ftp.unidata.ucar.edu/pub/netcdf/netcdf-${NETCDF_C_VERSION}.tar.gz -P /tmp \
    && tar -xf /tmp/netcdf-${NETCDF_C_VERSION}.tar.gz -C /tmp \
    && cd /tmp/netcdf-${NETCDF_C_VERSION} \
    && CPPFLAGS=-I${H5DIR}/include LDFLAGS=-L${H5DIR}/lib ./configure --prefix=/usr/local \
    && cd /tmp/netcdf-${NETCDF_C_VERSION} \
    && make \
    && cd /tmp/netcdf-${NETCDF_C_VERSION} \
    && make install \
    && rm -rf /tmp/netcdf-${NETCDF_C_VERSION}

# install netCDF-Fortran
ENV LD_LIBRARY_PATH=${NCDIR}/lib
RUN NETCDF_F_VERSION="4.4.4" \
    && cd /tmp \
    && wget ftp://ftp.unidata.ucar.edu/pub/netcdf/netcdf-fortran-${NETCDF_F_VERSION}.tar.gz \
    && tar -xf netcdf-fortran-${NETCDF_F_VERSION}.tar.gz \
    && cd /tmp/netcdf-fortran-${NETCDF_F_VERSION} \
    && CPPFLAGS=-I${NCDIR}/include LDFLAGS=-L${NCDIR}/lib ./configure --prefix=${NFDIR} \
    && make \
    && make install \
    && cd / \
    && rm -rf /tmp/netcdf-fortran-${NETCDF_F_VERSION}

###################################
## create docker user
RUN useradd -ms /bin/bash docker
RUN usermod -aG sudo docker
RUN chmod -R 777 /home/docker/
############################

###WRF | WRF-Hydro and WPS

#Set WRF and WPS version argument
ARG WRF_VERSION="4.1.2"
ARG WPS_VERSION="4.1"

#Set WRF-Hydro version argument
ARG HYDRO_VERSION="5.1.1-beta"

#Install coupled WRF | WRF-Hydro AND WPS
WORKDIR /home/docker/WRF_WPS
#
# Download sources for versions specified by the WRFWPS_VERSION and HYDRO_VERSION arguments
#
RUN wget https://github.com/wrf-model/WRF/archive/v${WRF_VERSION}.tar.gz \
        && tar -zxf v${WRF_VERSION}.tar.gz \
        && mv WRF-${WRF_VERSION} WRF \
        && rm v${WRF_VERSION}.tar.gz
#RUN wget https://github.com/NCAR/wrf_hydro_nwm_public/archive/v${HYDRO_VERSION}.tar.gz \
#        && tar -zxf v${HYDRO_VERSION}.tar.gz \
#        && rm -r WRF/hydro \
#        && cp -r wrf_hydro_nwm_public*/trunk/NDHMS WRF/hydro \
#        && rm v${HYDRO_VERSION}.tar.gz
RUN git clone --single-branch --branch v5.1.1 https://github.com/NCAR/wrf_hydro_nwm_public.git \
        && rm -r WRF/hydro \
        && cp -r wrf_hydro_nwm_public*/trunk/NDHMS WRF/hydro 
RUN wget https://github.com/wrf-model/WPS/archive/v${WPS_VERSION}.tar.gz \
	&& tar -zxf v${WPS_VERSION}.tar.gz \
        && mv WPS-${WPS_VERSION} WPS \
	&& rm v${WPS_VERSION}.tar.gz

# Set paths to required libraries
ENV JASPERLIB=/usr/lib
ENV JASPERINC=/usr/include
ENV NETCDF=/usr/local

# Set WRF-Hydro environment variables
ENV WRF_HYDRO=1
ENV HYDRO_D=1
ENV SPATIAL_SOIL=0
ENV WRF_HYDRO_RAPID=0
ENV WRFIO_NCD_LARGE_FILE_SUPPORT=1
ENV WRF_HYDRO_NUDGING=0

# Build WRF first, required for WPS
WORKDIR /home/docker/WRF_WPS/WRF
RUN printf '34\n1\n' | ./configure \
	&& ./compile em_real  

# Build WPS second after WRF is built
WORKDIR /home/docker/WRF_WPS/WPS
RUN printf '1\n' | ./configure \
	&& ./compile 

RUN chmod -R 777 /home/docker/WRF_WPS

###################################
# Python
RUN wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh \
    && bash Miniconda3-latest-Linux-x86_64.sh -b -p /home/docker/miniconda3 \
    && rm Miniconda3-latest-Linux-x86_64.sh

#Set environment variables
ENV PATH="/home/docker/miniconda3/bin:${PATH}"

WORKDIR /home/docker

RUN conda install nodejs

# Install modules
RUN pip install --upgrade pip
RUN pip install numpy cython 
RUN pip install jupyterlab jupyter_contrib_nbextensions ipython h5py netcdf4 matplotlib dask toolz xarray

RUN jupyter labextension install @jupyterlab/toc
RUN jupyter labextension install @pyviz/jupyterlab_pyviz

RUN pip install xrviz

# Get jupyter config
COPY ./jupyter_notebook_config.py /home/docker/.jupyter/
RUN chmod -R 777 /home/docker/.jupyter

############################
# Get geog data from google drive
## Get download script
COPY gdrive_download.py gdrive_download.py
RUN chmod 777 gdrive_download.py

RUN python gdrive_download.py --file_id 1X71fdaSEJ5GWyNY2MDIy9cC6E7A0kihl --dest_file geog_conus.tar.gz \
	&& tar -xf geog_conus.tar.gz \
	&& rm geog_conus.tar.gz
RUN chmod -R 777 /home/docker/geog_conus
RUN mv /home/docker/geog_conus /home/docker/WRF_WPS/geog_conus 
RUN rm gdrive_download.py

####################################
#Get the entrypoint script to download the code release, start jupyter, etc
COPY ./entrypoint.sh /.
RUN chmod 777 /entrypoint.sh

RUN mkdir /home/docker/wrf-hydro-training/ && chmod -R 777 /home/docker/wrf-hydro-training/

USER docker
WORKDIR /home/docker

ENV SHELL=bash
ENTRYPOINT ["/entrypoint.sh"]
CMD ["interactive"]
