# rancher2-single-node-cluster-vagrant

A vagrant box which runs a Rancher 2.0 server and single node Kubernetes cluster for local development.

- Runs a local VirtualBox VM with hostname `rancher2` and IP address `192.168.88.100`. For convenience it's assumed `192.168.88.100 rancher2` has been added to the `hosts` file.
- Installs docker version 18.03.1-ce
- Starts a [Rancher 2.0](https://hub.docker.com/r/rancher/rancher/) server container and sets the server url to https://rancher2:8443/ where you can access the Rancher UI.
- Starts a [Rancher Agent](https://hub.docker.com/r/rancher/rancher-agent) container that will create a local Kubernetes cluster and link it to the Rancher Server. The API is accessible on https://rancher2:443/.
- Also contains  [Portainer](https://portainer.io/) container because I like to be able to see which containers Kubernetes is running. Accessible at http://rancher2:9000/.

First version was based on the Chef base box [bento/ubuntu-18.04](https://github.com/chef/bento/tree/master/ubuntu)
and included all the base images, but was 1.3G in size. Now using Alpine Linux for a smaller base (around 80 MB) and only pre-pulling
the Rancher 2.0 Server image which is around 160 MB compressed.

## Build

If you have [Packer](https://www.packer.io/) and [VirtualBox](https://www.virtualbox.org/) installed you can run `build.sh` to create a new box image.

Startup of images is done in the Vagrantfile inline provision script so that
fresh server tokens are generated at that time and not included in the box image.

Because of bug:
[workerPlane] Failed to bring up Worker Plane: Failed to start [kubelet] container on host [10.0.2.15]: Error response from daemon: linux mounts: path /var/lib/docker is mounted on /var/lib/docker but it is not a shared or slave mount

Add this line to pack.sh

sed -i '/ulimit -p unlimited/a mount --make-shared /' /etc/init.d/docker
```
#!/sbin/openrc-run
# Copyright 1999-2013 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2

command="${DOCKERD_BINARY:-/usr/bin/dockerd}"
pidfile="${DOCKER_PIDFILE:-/run/${RC_SVCNAME}.pid}"
command_args="-p \"${pidfile}\" ${DOCKER_OPTS}"
DOCKER_LOGFILE="${DOCKER_LOGFILE:-/var/log/${RC_SVCNAME}.log}"
DOCKER_ERRFILE="${DOCKER_ERRFILE:-${DOCKER_LOGFILE}}"
DOCKER_OUTFILE="${DOCKER_OUTFILE:-${DOCKER_LOGFILE}}"
start_stop_daemon_args="--background \
	--stderr \"${DOCKER_ERRFILE}\" --stdout \"${DOCKER_OUTFILE}\""

grsecdir=/proc/sys/kernel/grsecurity

depend() {
	need sysfs cgroups
}

start_pre() {
	checkpath -f -m 0644 -o root:docker "$DOCKER_LOGFILE"

	for i in $disable_grsec; do
		if [ -e "$grsecdir/$i" ]; then
			einfo " Disabling $i"
			echo 0 > "$grsecdir/$i"
		fi
	done
	ulimit -n 1048576

	# Having non-zero limits causes performance problems due to accounting overhead
	# in the kernel. We recommend using cgroups to do container-local accounting.
	ulimit -p unlimited
mount --make-shared /

	return 0
}
```

## Run

1. Add to local hosts file for convenient access
   ```
   192.168.88.100 rancher2
   ```
2. Install the Vagrant Alpine plugin `vagrant plugin install vagrant-alpine`
3. Use the supplied vagrant file and run `vagrant up`
4. Go to Rancher 2.0 and set the admin password: https://rancher2:8443/
5. Run `vagrant ssh` to log into the VM, then run `kubectl cluster-info`.
6. Once Portainer has been provisioned you can set the admin password: http://rancher2:9000/#/init/admin. Choose "Manage the local Docker environment" and connect.
   Sometimes I have to log out and in again after setting up the password.
7. Go to "Clusters" in the Rancher UI and see the state of the "local-cluster" which will
   probably take a while to download all the images and start the cluster.
   
## Setup kubectl
```
$ scp -i .vagrant/machines/default/virtualbox/private_key  -P 2222 vagrant@127.0.0.1:.kube/config .
$ sed -i -e 's/localhost/rancher2/g' config
$ export KUBECONFIG=./config 
$ kubectl get node
NAME       STATUS   ROLES                      AGE   VERSION
rancher2   Ready    controlplane,etcd,worker   44m   v1.10.5
$ kubectl run hello-node --image=gcr.io/hello-minikube-zero-install/hello-node --port=8080
$ vagrant ssh
rancher2:~$ wget https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin
/linux/amd64/kubectl
rancher2:~$ chmod +x kubectl 
rancher2:~$ sudo cp kubectl /usr/local/bin
rancher2:~$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-5f76cf6ccf-ppdmx   1/1     Running   0          18m

rancher2:~$ wget https://github.com/rancher/cli/releases/download/v2.0.5/rancher-linux-amd64-v2.0.5.tar.gz
rancher2:~$ tar -zxvf rancher-linux-amd64-v2.0.5.tar.xz 
rancher2:~$ sudo cp rancher-v2.0.5/rancher /usr/local/bin

```

## Useful references

- Alpine sections based on Matt Maier's [vagrant-alpine plugin](https://github.com/maier/vagrant-alpine), [alpine-vagrant 3.8 box](https://github.com/rgl/alpine-vagrant) and [packer-templates](https://github.com/maier/packer-templates/)
- https://ketzacoatl.github.io/posts/2017-06-01-use-existing-vagrant-box-in-a-packer-build.html describes how to build using another vagrant box as base, this build originally used https://vagrantcloud.com/bento/boxes/ubuntu-18.04/versions/201807.12.0/providers/virtualbox.box
- see https://gist.github.com/superseb/29af10c2de2a5e75ef816292ef3ae426 for example of Rancher 2 REST API calls to create and add a new cluster
- see https://gist.github.com/superseb/cad9b87c844f166b9c9bf97f5dea1609 for example of Rancher 2 REST API calls to create kubeconfig

# rancher2-multi-node-cluster-vagrant

```
Vagrantfile.multi fixes

Fix1: vb.customize ["modifyvm", :id, "--audio", "none"]

because of 

There was an error while executing `VBoxManage`, a CLI used by Vagrant
for controlling VirtualBox. The command and stderr is shown below.

Command: ["startvm", "d80ecd39-65eb-42af-96e2-efc692026af0", "--type", "headless"]

Stderr: VBoxManage: error: The specified string / bytes buffer was to small. Specify a larger one and retry. (VERR_CFGM_NOT_ENOUGH_SPACE)
VBoxManage: error: Details: code NS_ERROR_FAILURE (0x80004005), component ConsoleWrap, interface IConsole

Fix2:
config.vbguest.auto_update = false
config.vm.synced_folder ".", "/vagrant", disabled: true
    
```
```
$ cp Vagrantfile.multi Vagrantfile
$ vagrant up 

$cat /etc/hosts

## vagrant-hostmanager-start id: ccc6ca0f-3f4f-4911-89df-62b5159620b1
192.168.34.10	master.rancher.local
192.168.34.11	node1.rancher.local

## vagrant-hostmanager-end


Login browser: https://master.rancher.local
user:admin
password: password

Go to playground cluster -> Kubeconfig file
 
Copy configuration and put to config.multi file
VMhost$ export KUBECONFIG=./config.multi
VMhost$ kubectl get nodes
NAME    STATUS   ROLES                      AGE   VERSION
node1   Ready    controlplane,etcd,worker   34m   v1.10.5

```
