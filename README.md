# OpenShift Upgrade via ose-ansible 
The set of files to upgrade OpenShift, to 3.10 and 3.11, automatically.

# Setup Steps
## Setup local Git repo
Grab Docker files from the openshift-upgrade repo (and "change working directory" to it - all commands are to be ran in the
'openshift-upgrade' clone directory)
```git clone https://github.com/dfwnetman/openshift-upgrade.git
cd openshift-upgrade
```


##Set Environment Variables
Set OCP_VER environment variable to the version, ('3.10' or '3.11'), being upgraded (NOTE: the version is used for building & running the docker container).
```export OCP_VER=<openshift_major_dot_minor_version>
For example, for 3.10:
export OCP_VER=3.10

For example, for 3.11:
export OCP_VER=3.11
```

Set the HOSTS_FILE environment variable to the ansible hosts inventory file
```export HOSTS_FILE=ansible-inventory/<ansible_hosts_inventory_file>
For example:
export HOSTS_FILE=ansible-inventory/my_hosts
```
# Docker Build Steps
Build ose-ansible docker image
```GROUP="$(id -ng)"
GID="$(id -g)"
UID="$(id -u)"
sudo docker build --build-arg GID=$GID --build-arg GROUP=$GROUP --build-arg
UID=$UID --build-arg USER=$USER -t ose-ansible-$OCP_VER -f
Dockerfile_$OCP_VER .
```

# Docker Run Steps
## Auto Upgrade
Spin up ose-ansible-<ocp_ver> docker container, in 'auto' mode
```UID="$(id -u)"
sudo docker run --name ose-ansible -u $uid -dit -v
$PWD/ansible-inventory/:/opt/app-root/src/ansible-inventory:Z -v
$HOME/.ssh/id_rsa:/opt/app-root/src/.ssh/id_rsa:Z ose-ansible-$OCP_VER
auto ${HOSTS_FILE}
```

## Interactive Upgrade
Spin up ose-ansible-<ocp_ver> docker container, in 'interactive' mode
```UID="$(id -u)"
sudo docker run --name ose-ansible-$OCP_VER -u $UID -dit -v
$PWD/ansible-inventory/:/opt/app-root/src/ansible-inventory:Z -v
$HOME/.ssh/id_rsa:/opt/app-root/src/.ssh/id_rsa:Z ose-ansible-$OCP_VER
```

### Pre-Installation Steps
```sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -check-hosts-file -m
prep_verify_ansible_hosts
sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m prep_pull_docker_images
sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m prep_verify_sbin_in_path
sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m prep_verify_local_file_systems
sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m prep_test_conn_local_yum_repo
sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m prep_local_yum_repo
sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m prep_verify_local_yum_repo
sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m prep_backup
sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m prep_install_excluder_packages
```
### Upgrade Steps

Upgrade Control Plane
```sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m upgrade_control_plane
```
Upgrade Infra Nodes
```sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m upgrade_infra_nodes
```
Upgrade App/Compute Nodes, in chunks (up to # ansible 'forks', 20)
```sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m upgrade_app_nodes \
[-b <begin_node>] [-e <end_node>] [-n <openshift_upgrade_nodes_serial>]
```

Upgrade metrics
```sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m upgrade_metrics
```
Post-Upgrade Steps
```sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m
post_verify_router_registry_and_metrics_pods
sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m post_restore_yum_repo
```
### Some pre-built "repair" Steps
```sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m fix_rsyslog_logging \
-h <FQDN_OF_SINGLE_HOST_TO_BOUNCE_RSYSLOG_AND_JOURNALD>

sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m fix_remove_label_from_nodes_in_range
[-b <begin_node>] [-e <end_node>] [-n <openshift_upgrade_nodes_serial>]

sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m fix_schedule_nodes_in_range
[-b <begin_node>] [-e <end_node>] [-n <openshift_upgrade_nodes_serial>]

sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m fix_remove_label_from_all_nodes
sudo docker exec -t ose-ansible-$OCP_VER ./upgrade_ocp.sh -i
${HOSTS_FILE} -m fix_schedule_all_nodes
```
