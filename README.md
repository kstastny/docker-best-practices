# Docker Best Practices

_Documentation of best practices for working with Docker according to my personal preferences. Please note that I use Windows for development, therefore the documentation is suited towards that end._


## Installation

Since Docker for Desktop comes with many issues (licensing being one of them, stability another), I prefer running Docker in WSL.

### Setup WSL distribution
To install WSL, see https://docs.microsoft.com/en-us/windows/wsl/install

1. Install Ubuntu

    `wsl --install -d Ubuntu`

1. After installation, login to the new distribution and choose username and password. The prompt should open automatically.
1. Update and upgrade packages

    `sudo apt update && sudo apt upgrade`


### Install Docker

1. Install pre-requisities

    `sudo apt install --no-install-recommends apt-transport-https ca-certificates curl gnupg2`

1. Trust the docker repo with packages
    1. set temporary OS-specific variables
        
        `source /etc/os-release`
    1. Trust the repo
        
        `curl -fsSL https://download.docker.com/linux/${ID}/gpg | sudo apt-key add -`
    1. Add and update the repo information

        `echo "deb [arch=amd64] https://download.docker.com/linux/${ID} ${VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/docker.list`  
        `sudo apt update`

1. Install Docker itself
    `sudo apt install docker-ce docker-ce-cli containerd.io docker-compose`

Now, you should configure the user permissions and setup docker daemon to autorun but since I tend to launch docker daemon manually, I will skip this step and just point you to [https://dev.to/bowmanjd/install-docker-on-windows-wsl-without-docker-desktop-34m9](Jonathan Bowman's tutorial)


To verify the installation

1. run docker deamon `sudo dockerd`
1. run hello world container `docker run --rm hello-world`

This will print "Hello from Docker" and some informational messages if the installation is correct.


### Sources

1. [https://dev.to/bowmanjd/install-docker-on-windows-wsl-without-docker-desktop-34m9](https://dev.to/bowmanjd/install-docker-on-windows-wsl-without-docker-desktop-34m9)
1. [https://dev.to/_nicolas_louis_/how-to-run-docker-on-windows-without-docker-desktop-hik](https://dev.to/_nicolas_louis_/how-to-run-docker-on-windows-without-docker-desktop-hik)


## How to Change Location of Virtual Hard Drives

Since I have a small system disk, I prefer moving the VHDs to a different harddrive to save space on the system.

1. Shutdown the WSL image. Let's assume that its name is "Ubuntu"
   
    `wsl --shutdown`

1. Export the image data

    `wsl --export Ubuntu "D:\DockerDesktop-vm-data\Ubuntu.tar"`

1. Unregister the image. Note that the ext4.vhdx will be removed automatically.

    `wsl --unregister Ubuntu`

1. Import the WSL image to different directory.

    `wsl --import Ubuntu "D:\DockerDesktop-vm-data\wsl" "D:\DockerDesktop-vm-data\Ubuntu.tar" --version 2`

1. Run the WSL distribution and verify. You may now remove the `Ubuntu.tar` file    


### Source

https://stackoverflow.com/questions/62441307/how-can-i-change-the-location-of-docker-images-when-using-docker-desktop-on-wsl2