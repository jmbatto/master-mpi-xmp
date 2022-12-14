FROM debian:bullseye-slim
# an image with OpenMPI4.1, used without root privileges for sshd
# XMP and telegraf
# ------------------------------------------------------------
# Do basic install
# ------------------------------------------------------------
RUN apt-get update \
    && mkdir -p /usr/share/man/man1 \
    && apt-get install -y gcc ssh wget vim curl net-tools bison flex openjdk-11-jdk-headless autoconf make libtool m4 automake bzip2 libxml2 libxml2-dev gfortran g++ iputils-ping pkg-config colordiff nano git sudo lsof gawk emacs jq neofetch libtdl* astyle cmake gdb strace binutils-dev dnsutils netcat libgomp1 googletest supervisor \
    && adduser --uid 1000 --home /home/mpiuser --shell /bin/bash \
       --disabled-password --gecos '' mpiuser \
    && passwd -d mpiuser \
    && apt-get install -y openssh-server \
    && mkdir -p /run/sshd /home/mpiuser/.ssh /home/mpiuser/.ssh-source \
    && echo "StrictHostKeyChecking no" > /home/mpiuser/.ssh/config \
    && chown -R mpiuser /home/mpiuser \
    && sed -i s/#PermitRootLogin.*/PermitRootLogin\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PubkeyAuthentication.*/PubkeyAuthentication\ no/ /etc/ssh/sshd_config \
    && sed -i s/.*UsePAM.*/UsePAM\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PasswordAuthentication.*/PasswordAuthentication\ yes/ /etc/ssh/sshd_config \
    && sed -i s/#PermitEmptyPasswords.*/PermitEmptyPasswords\ yes/ /etc/ssh/sshd_config \
    && sed -i s/#ChallengeResponse.*/ChallengeResponseAuthentication\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PermitUserEnvironment.*/PermitUserEnvironment\ yes/ /etc/ssh/sshd_config \
	&& adduser mpiuser sudo

ENV PREFIX=/usr/local \
	OPENMPI_VERSION=4.1.4 \
    LD_LIBRARY_PATH=/usr/local/lib \
    DEBCONF_NOWARNINGS=yes

# ------------------------------------------------------------
# Install OpenMPI 4.1
# https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.4.tar.gz
# ------------------------------------------------------------

# OpenMPI v4.1
RUN repo="https://download.open-mpi.org/release/open-mpi/v4.1" \
    && curl --location --silent --show-error --output openmpi.tar.gz \
      "${repo}/openmpi-${OPENMPI_VERSION}.tar.gz" \
    && tar xzf openmpi.tar.gz -C /tmp/ \
    && cd /tmp/openmpi-${OPENMPI_VERSION} \
	&& env CFLAGS="-O2 -std=gnu99 -fopenmp" \
    && ./configure --prefix=${PREFIX} \
    && make \
    && make install \
    && ldconfig \
    && cd / \
    && rm -rf /tmp/openmpi-${OPENMPI_VERSION} /home/mpiuser/openmpi.tar.gz

# ------------------------------------------------------------
# Add some parameters for MPI, mpishare - a folder shared through the nodes
# ------------------------------------------------------------	
RUN mkdir -p /usr/local/var/mpishare

RUN chown -R 1000:1000 /usr/local/var/mpishare

RUN echo "mpiuser ALL=(ALL) NOPASSWD:ALL\n" >> /etc/sudoers

RUN rm -fr /home/mpiuser/.openmpi && mkdir -p /home/mpiuser/.openmpi
RUN cd /home/mpiuser/.openmpi \
	&& echo "btl = tcp,self \n" \
	"btl_tcp_if_include = eth0 \n" \
	"plm_rsh_no_tree_spawn = 1 \n" >> default-mca-params.conf

RUN chown -R 1000:1000 /home/mpiuser/.openmpi

RUN echo "rmaps_base_oversubscribe = 1\n" >> /usr/local/etc/openmpi-mca-params.conf
RUN echo "rmaps_base_inherit = 1\n" >> /usr/local/etc/openmpi-mca-params.conf


# ------------------------------------------------------------
# Start mpi python install / user mpiuser
# ------------------------------------------------------------
RUN apt-get install -y --no-install-recommends python3-dev python3-numpy python3-pip python3-virtualenv python3-scipy 2to3 \
    && apt-get clean && apt-get purge && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

RUN pip install --upgrade pip

# in order to have python related to mpiuser account
USER mpiuser
RUN  pip install --user -U setuptools \
    && pip install --user mpi4py
USER root


# ------------------------------------------------------------
# Copy MPI4PY example scripts
# ------------------------------------------------------------



ADD ./mpi4py_benchmarks /home/mpiuser/mpi4py_benchmarks
RUN chown -R mpiuser:mpiuser /home/mpiuser/mpi4py_benchmarks
RUN cd /home/mpiuser/mpi4py_benchmarks && 2to3 -w --no-diffs *.py



# ------------------------------------------------------------
# Load omni-compiler 1.3.4 from a public repo
# ------------------------------------------------------------
RUN mkdir -p /home/mpiuser/YMLEnvironment
WORKDIR /home/mpiuser/YMLEnvironment
RUN wget --no-check-certificate --content-disposition https://omni-compiler.org/download/stable/omnicompiler-1.3.4.tar.bz2
RUN bunzip2 omnicompiler-1.3.4.tar.bz2 \
	&& tar xvf omnicompiler-1.3.4.tar \
	&& rm /home/mpiuser/YMLEnvironment/omnicompiler-1.3.4.tar
	
# ------------------------------------------------------------
# Install omnicompiler-1.3.4 : requested javac
# ------------------------------------------------------------

WORKDIR /home/mpiuser/YMLEnvironment/omnicompiler-1.3.4
RUN cd /home/mpiuser/YMLEnvironment/omnicompiler-1.3.4 \
	&& export FCFLAGS="-w -fallow-argument-mismatch -O2" \
	&& export FFLAGS="-w -fallow-argument-mismatch -O2" \
	&& export MPI_FCFLAGS="-fopenmp -fallow-argument-mismatch -O2" \	
	&& export CPPFLAGS="-fallow-argument-mismatch -DOMNI_CPU_X86_64 -DOMNI_OS_LINUX -DGNU_SOURCE -D_REENTRANT" \
	&& ./configure --prefix=${PREFIX} --with-libxml2=/usr \
	&& make && make install && make clean && ldconfig

# ------------------------------------------------------------
# Install test-spawn-xmp
# ------------------------------------------------------------
USER mpiuser
COPY test-spawn-xmp /home/mpiuser/YMLEnvironment/test-spawn-xmp
USER root
RUN chown -R mpiuser:mpiuser /home/mpiuser/YMLEnvironment

# ------------------------------------------------------------
# Install telegraf
# ------------------------------------------------------------

RUN apt-get update && apt-get install -y --no-install-recommends gpgv ca-certificates apt-transport-https gnupg lsb-release
RUN wget -qO- https://repos.influxdata.com/influxdb.key | apt-key add -
RUN echo "deb https://repos.influxdata.com/debian $(lsb_release -cs) stable"| tee /etc/apt/sources.list.d/influxdb.list
RUN apt-get update && apt-get install -y --no-install-recommends telegraf && apt-get clean && apt-get purge && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*


# ------------------------------------------------------------
# The .ssh-source dir contains RSA keys - put in place with docker-compose
# ------------------------------------------------------------


RUN touch /home/mpiuser/.ssh-source/authorized_keys
RUN touch /home/mpiuser/.ssh-source/id_rsa


# ------------------------------------------------------------
# Do SSHd parameter to enable mpiuser to run it
# ------------------------------------------------------------
RUN sed -i s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation\ no/ /etc/ssh/sshd_config
RUN mkdir -p /home/mpiuser/ssh
RUN ssh-keygen -q -N "" -t dsa -f /home/mpiuser/ssh/ssh_host_dsa_key \
	&& ssh-keygen -q -N "" -t rsa -b 4096 -f /home/mpiuser/ssh/ssh_host_rsa_key \
	&& ssh-keygen -q -N "" -t ecdsa -f /home/mpiuser/ssh/ssh_host_ecdsa_key \
	&& ssh-keygen -q -N "" -t ed25519 -f /home/mpiuser/ssh/ssh_host_ed25519_key

RUN cp /etc/ssh/sshd_config /home/mpiuser/ssh/

RUN sed -i s/#HostKey\ \\/etc\\/ssh/HostKey\ \\/home\\/mpiuser\\/ssh/ /home/mpiuser/ssh/sshd_config
RUN sed -i s/#PidFile\ \\/var\\/run/PidFile\ \\/home\\/mpiuser\\/ssh/ /home/mpiuser/ssh/sshd_config
RUN sed -i s/#LogLevel.*/LogLevel\ DEBUG3/ /home/mpiuser/ssh/sshd_config
RUN sed -i s/PubkeyAuthentication\ no/PubkeyAuthentication\ yes/ /home/mpiuser/ssh/sshd_config

RUN chown -R mpiuser:mpiuser /home/mpiuser/ssh

# ------------------------------------------------------------
# supervisord to run entrypoint
# ------------------------------------------------------------
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# ------------------------------------------------------------
# telegraf to run entrypoint
# ------------------------------------------------------------
COPY telegraf.conf /etc/telegraf/telegraf.conf

EXPOSE 22 80 8125

WORKDIR /home/mpiuser

CMD cp -R /home/mpiuser/.ssh-source/* /home/mpiuser/.ssh \
	&& chmod 700 /home/mpiuser/.ssh/authorized_keys \
	&& chmod 700 /home/mpiuser/.ssh/id_rsa \
	&& chown -R mpiuser:mpiuser /home/mpiuser/.ssh \
	&& exec /usr/bin/supervisord -c /etc/supervisor/conf.d/supervisord.conf
#	&& exec /usr/sbin/sshd -D -f /home/mpiuser/ssh/sshd_config -E /tmp/sshd.log
# do not detach (-D), log to stderr (-e)
# CMD ["/usr/sbin/sshd", "-D", "-e"]