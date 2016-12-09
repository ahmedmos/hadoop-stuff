# How to run HDP 2.5 sandbox as a Docker container _ON WINDOWS 10_?

I'm going to show quickly how can you import and launch HDP 2.5 sandbox to Docker on Windows 10. I just tried it and it surprisingly worked (as I'm not really a Windows guy), and I actually like this much better than spinning up a whole VM :)

## First thing:
Download the HDP dockerized image

## Enable Windows capabilities
First, enable the necessary capabilities in your Windows 10. You basically need to enable Hyper-V and the Containers capability.
- Press ```Ctrl+X``` and choose _Programs and Features_
- On the left, click on _Turn Windows Features On/Off_
- Make sure Hyper-V and Containers are selected

Actually, with Hyper-V only, you can create your own VMs without the need to install VirtualBox or VMWare. It's pretty cool and very lightweight; but unfortunately there's no ready HDP or Cloudera VMs ported for it, and I'm no expert on converting the formats (I don't have time for that atcually!).

## Install & Configure Docker
Go to the official [Docker Windows](https://docs.docker.com/docker-for-windows/) page and download and install the stable version. 

Once installed you can use docker natively from _PowerShell_ and the command prompt (I used _PowerShell_ because it has a linux-like feeling that I'm comfortable with).

Now, run **PowerShell in "elevated mode"** and do the following: (_NOTE: the following is copied from Docker's website to make things simple for the reader_)
- run ```docker version```
- run ```docker run hello-world``` to make sure docker can pull a simple image and containerize it and run it
- run ```docker rm hello-world``` to remove the container
- run ```docker rmi hello-world``` to remove the image

Now, run the following to have auto completion for docker commands in PowerShell:
- run ```Install-Module -Scope CurrentUser posh-docker -Force```
- run ```Set-ExecutionPolicy RemoteSigned```
- run ```Import-Module posh-docker```
- run ``` Add-Content $PROFILE "`nImport-Module posh-docker"```

If the posh-docker didn't work, you might be missing a module, just Google (or Bing!) it, you're now basically ready for the real stuff :)

## Import HDP Sandbox into Docker
- I assume you already downloaded the HDP dockerized image
- run the script above; it basically summarizes the installation steps mentioned on HDP's installation guide.

## Make sure things are running
The script makes sure the container is running along with all the services underneath.
- Open [http://localhost:8888](http://localhost:8888)
- Open [http://localhost:8080](http://localhost:8080) and login using ```maria_dev/maria_dev```
- run ```docker exec -ti sandbox bash``` to get inside the container and check the running services.

## Stopping & Restarting 
- You can stop your container using ```docker stop sandbox```
- Run the same script again and it will restart everything for you, use ```https://raw.githubusercontent.com/ahmedmos/hadoop-stuff/master/dockerized-hdp2.5-windows/load-HDP25-Docker-Win.bat -OutFile runHDPDocker.ps``` and run the downloaded file.
