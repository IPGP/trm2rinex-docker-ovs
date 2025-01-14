## Introduction 

This is the IPGP-OVS's maintened fork of the Docker+Wine (`trm2rinex`) wrapper for Trimble `ConvertToRinex` tool  🛰️ 🌐 🐋 🍷  

The original repository made by _Matioupi_ is available at: https://github.com/Matioupi/trm2rinex-docker  
(Many thanks to him!)

From now on (September 2024), we must fork the original repository, because it is not maintained anymore and related software has evolved.

Maintainer of this fork: Pierre Sakic (sakic@ipgp.fr)

See also the README of the original repository for more details: [readme_original.md](readme_original.md)

## Changelog
* 2025-10-14: 
  * Make the docker compatible ẃith a Debian 12.  
  * Make the docker compilation faster, no needs to clone the whole wine repository anymore.
* 2025-01-03: better handling of the user & group ID
* 2024-10-29: 
  * Back to `ConvertToRinex` v3.14.  
  * For some files, v3.15 leads to error:  
`t01dll assembly:<unknown assembly> type:<unknown type> member:(null)`
* 2024-09-07: Change the wine repository for https://gitlab.winehq.org/wine/wine.git
* 2024-09-06: Change for `ConvertToRinex` v3.15

## Installation

This memo is about installing the `trm2rinex` software in a Docker container.   
The software is a Windows software, so it is run in a Wine environment.

### Prerequisite: install Docker 
#### on Ubuntu
```
https://www.simplilearn.com/tutorials/docker-tutorial/how-to-install-docker-on-ubuntu
```
#### on Debian
```
https://linuxiac.com/how-to-install-docker-on-debian-12-bookworm/
```

### Create Docker group
```
sudo addgroup --system docker
sudo adduser $USER docker
newgrp docker
```

### Clone the IPGP-OVS forked repository
```
cd $HOME/your/favorite/folder
git clone https://github.com/IPGP/trm2rinex-docker-ovs
```

original repository (for indicative legacy purposes):
```
git clone https://github.com/Matioupi/trm2rinex-docker
```

### Build
```
cd trm2rinex-docker-ovs
docker build --build-arg USER_UID=$(id -u) -t trm2rinex:cli-light .
```
See troubleshooter for the user & group ID question

### Clean useless Docker images
at the end of the installation
```
docker image ls
```
and remove intermediate images
```
docker image rm 925d947bf045
```

## Usage

### Run the test file conversion
```
cd <...>/trm2rinex-docker
docker run --rm -v "$(pwd):/data" trm2rinex:cli-light data/MAGC320b.2021.rt27 -p data/out
```
## Troubleshooter

### Issue about removing folders during Docker compilation
#### Error
In the log, at the step 25/65, you have:
```
rm: cannot remove '/opt/wine/share/include/windows': Directory not empty
```
#### Solution
then in the *Dockerfile*, comment the lines 170-173:
```
RUN strip -s ${WINE_INSTALL_PREFIX}/lib/wine/i386-windows/* \
    && strip -s ${WINE_INSTALL_PREFIX}/lib/wine/i386-unix/* \
    && strip -s ${WINE_INSTALL_PREFIX}/bin/wine \
                ${WINE_INSTALL_PREFIX}/bin/wineserver \
                ${WINE_INSTALL_PREFIX}/bin/wine-preloader
#    && rm -rf ${WINE_INSTALL_PREFIX}/include \
#    && rm -rf ${WINE_INSTALL_PREFIX}/lib/wine/i386-unix/*.a \
#    && rm -rf ${WINE_INSTALL_PREFIX}/lib/wine/i386-windows/*.a \
#    && rm -rf ${WINE_INSTALL_PREFIX}/share/man
```

and lines 232-234:
```
USER root
COPY clean.sh /home/${USER_NAME}/clean.sh
RUN chmod 755 /home/${USER_NAME}/clean.sh
#RUN rm -rf /home/${USER_NAME}/.wine/drive_c/windows/Installer \
#    && rm -rf /tmp/* \
#    && /home/${USER_NAME}/clean.sh ${USER_NAME} ${WINE_INSTALL_PREFIX}
```

### Issue about converting the files
#### Error 
```
user@computer:~/SOFT/trm2rinex-docker$ docker run --rm -v "$(pwd):/data" trm2rinex:cli-light data/MAGC320b.2021.rt27 data/out
Scanning data/MAGC320b.2021.rt27...Complete!
Converting MAGC320b.2021.rt27...Error: CrinexFile - System.UnauthorizedAccessException: Access to the path "Z:\data\MAGC320b.2021.21n" is denied.
  at System.IO.FileStream..ctor (System.String path, System.IO.FileMode mode, System.IO.FileAccess access, System.IO.FileShare share, System.Int32 bufferSize, System.Boolean anonymous, System.IO.FileOptions options) [0x0019e] in <bee01833bcad4475a5c84b3c3d7e0cd6>:0 
  at System.IO.FileStream..ctor (System.String path, System.IO.FileMode mode, System.IO.FileAccess access, System.IO.FileShare share, System.Int32 bufferSize, System.IO.FileOptions options) [0x00000] in <bee01833bcad4475a5c84b3c3d7e0cd6>:0 
  at (wrapper remoting-invoke-with-check) System.IO.FileStream..ctor(string,System.IO.FileMode,System.IO.FileAccess,System.IO.FileShare,int,System.IO.FileOptions)
  at System.IO.StreamWriter..ctor (System.String path, System.Boolean append, System.Text.Encoding encoding, System.Int32 bufferSize) [0x00055] in <bee01833bcad4475a5c84b3c3d7e0cd6>:0 
  at System.IO.StreamWriter..ctor (System.String path, System.Boolean append) [0x00008] in <bee01833bcad4475a5c84b3c3d7e0cd6>:0 
  at (wrapper remoting-invoke-with-check) System.IO.StreamWriter..ctor(string,bool)
  at System.IO.File.CreateText (System.String path) [0x0000e] in <bee01833bcad4475a5c84b3c3d7e0cd6>:0 
  at trimble.rinex.CrinexFile..ctor (System.String inputFilePath, System.IO.FileMode mode) [0x00176] in <6b991ac620904a08a4ad53d3cafee6d1>:0 : Z:\data\MAGC320b.2021.21n using mode: Create
```
#### Solution 1
The problem occurs because the output file is about to be created in a subfolder `out` within the current directory (`$(pwd)`), but this `out` folder does not exists or is not writable. Thus a simple
```commandline
mkdir out
```
within the current/working directory should solve the problem.    

#### Solution 2
You will continue to face this error when you try to mount two volumes, one for the input (`inp`), the other for the output (`out`), and if one bound folder is included in the other, e.g.:
```
-v ${PWD}:/inp -v ${PWD}/converted:/out
```
Then, define your `out` bound volume somewhere else:
```
-v ${PWD}:/inp -v /a/different/place/converted:/out
```

#### Solution 3
Give full access right to your `out` folder
```
chmod 777 out
```

### Change user and group ID
This is a facultative step, but we recommend to change the `USER_UID` & `USER_GID` when building the docker

#### Solution 1
```
docker build --build-arg USER_UID=$(id -u) -t trm2rinex:cli-light .
```
However, adding the group with `--build-arg USER_GID=$(id -g)` is not recommended

#### Solution 2
Change the values defined in the `Dockerfile` preamble

* get the right user & group ID values:
```
id -u
```
* In the `Dockerfile` change the values of:
```
ARG USER_UID=1000
```
However, changing the group `ARG USER_GID=100` with `id -g` is not recommended

### Deamon socket permission denied
#### Error
```
ERROR: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Head "http://%2Fvar%2Frun%2Fdocker.sock/_ping": dial unix /var/run/docker.sock: connect: permission denied
```

#### Solution
Check if your user is correctly added to the `docker` group
https://stackoverflow.com/questions/48957195/how-to-fix-docker-got-permission-denied-issue

###
#### Error
While building the Docker image, you have the following error:
```commandline
RPC failed; curl 56 GnuTLS recv error (-9): Error decoding the received TLS packet.
```

#### Solution
Your connexion is too slow, and a timeout occurs while cloning the git wine repository.  
Follow these solutions to clone the wished git tag only and increase the timeout.  
_It is implemented per default in the Dockerfile_.
https://stackoverflow.com/questions/6842687/the-remote-end-hung-up-unexpectedly-while-git-cloning
https://stackoverflow.com/questions/38378914/how-to-fix-git-error-rpc-failed-curl-56-gnutls




