# django-demo

Use ansible to create a Django 5 environment on top of a bootc based OS with the correct python 3 pacakges to support it


The bootc container only contains the OS level mods required for this Django layer to become a functional app environment once configured at run time

**note: the separation of the playbooks is ongoing**

The first set of ansible playbooks contains 'generic' customizations that wire up the django environment according to local standards in prepartion for landing the application later

The second set of ansible playbooks come from a represent run time deployment of the application specific configurations

On the **build host** you'll need the following packages:
* podman
* ansible
* general, podman, and postgresql ansible collections


```
sudo dnf install -y podman ansible

ansible-galaxy collection install containers.podman
ansible-galaxy collection install community.general
ansible-galaxy collection install community.postgresql
```

Once you have the prerequisites, from the root of this repo you can build and run the container

```
podman build -t bootc-django-t1 bootc/
podman run -d --name atest bootc-django-t1 /sbin/init
```

Run the site playbook to make all the config changes to prep this as a deployable django environment ready for an application. The selinux context setting is being worked on, but should get autorelabeled when built by image builder.

```
ansible-playbook -c podman -i ansible/inventory.containers ansible/site.yml
```

Once the playbook completes, we have the container configured for our django standards. There aren't any exposed ports in this config, but you can confirm the skeleton is working from a bash shell. We can convert the container back to an image, then build our QCOW2 image for use.

```
podman exec -it atest /bin/bash
bash-5.2# systemctl status nginx postgresql gunicorn.socket
bash-5.2# curl --unix-socket /run/gunicorn.sock localhost

podman stop atest
podman commit atest localhost:5000/bootc-django-conf
podman push localhost:5000/bootc-django-conf
```

If needed you can create any new configs needed for the qcow before running the build, like the interactive user.  Create the target directory for the image builder artifacts, then run the image builder container. 

```
mkdir output

sudo podman run --rm -it --privileged --pull=newer --network=host --security-opt label=type:unconfined_t  -v $(pwd)/config.json:/config.json  -v $(pwd)/output:/output  quay.io/centos-bootc/bootc-image-builder:latest --tls-verify=false --type qcow2   --config /config.json localhost:5000/bootc-django-conf
```

Get the QCOW image from the output/qcow directory and import into a KVM environment to test.

**VM Issues**

The services are not enabled on boot in the QCOW image
Seboolean needs to be set after first boot to allow communication over the unix socket

```
setsebool -P daemons_enable_cluster_mode true

systemctl enable --now postgresql nginx.service gunicorn.socket

```