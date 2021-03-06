# Installing AEP from dev puddles using ansible

* [Requirements](#requirements)
* [Caveats](#caveats)
* [Known Issues](#known-issues)
* [Configuring the host inventory](#configuring-the-host-inventory)
* [Creating the default variables for the hosts and host groups](#creating-the-default-variables-for-the-hosts-and-host-groups)
* [Running the ansible playbooks](#running-the-ansible-playbooks)
* [Post-ansible steps](#post-ansible-steps)
* [Overriding detected ip addresses and hostnames](#overriding-detected-ip-addresses-and-hostnames)

## Requirements
* ansible
  * Tested using ansible 1.9.1 and 1.9.2
  * There is currently a known issue with ansible-1.9.0, you can downgrade to 1.8.4 on Fedora by installing one of the builds from Koji: http://koji.fedoraproject.org/koji/packageinfo?packageID=13842
  * Available in Fedora channels
  * Available for EL with EPEL and Optional channel
* One or more RHEL 7.1 VMs
* Either ssh key based auth for the root user or ssh key based auth for a user
  with sudo access (no password)
* A checkout of atomic-enterprise-ansible from https://github.com/projectatomic/atomic-enterprise-ansible/

  ```sh
  git clone https://github.com/projectatomic/atomic-enterprise-ansible.git
  cd atomic-enterprise-ansible
  ```

## Caveats
This ansible repo is currently under heavy revision for providing OSE support;
the following items are highly likely to change before the OSE support is
merged into the upstream repo:
  * the current git branch for testing
  * how the inventory file should be configured
  * variables that need to be set
  * bootstrapping steps
  * other configuration steps

## Known Issues
* Host subscriptions are not configurable yet, the hosts need to be
  pre-registered with subscription-manager or have the RHEL base repo
  pre-configured. If using subscription-manager the following commands will
  disable all but the rhel-7-server rhel-7-server-extras and
  rhel-server7-ose-beta repos:
```sh
subscription-manager repos --disable="*"
subscription-manager repos \
--enable="rhel-7-server-rpms" \
--enable="rhel-7-server-extras-rpms" \
--enable="rhel-server-7-ose-beta-rpms"
```
* Configuration of router is not automated yet
* Configuration of docker-registry is not automated yet

## Configuring the host inventory
[Ansible docs](http://docs.ansible.com/intro_inventory.html)

Example inventory file for configuring one master and two nodes for the test
environment. This can be configured in the default inventory file
(/etc/ansible/hosts), or using a custom file and passing the --inventory
option to ansible-playbook.

/etc/ansible/hosts:
```ini
# This is an example of a bring your own (byo) host inventory

# Create an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters
nodes

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
# SSH user, this user should allow ssh based auth without requiring a password
ansible_ssh_user=root

# If ansible_ssh_user is not root, ansible_sudo must be set to true
#ansible_sudo=true

# To deploy origin, change deployment_type to origin
deployment_type=enterprise

# Pre-release registry URL
oreg_url=docker-buildvm-rhose.usersys.redhat.com:5000/openshift3/ose-${component}:${version}

# Pre-release additional repo
openshift_additional_repos=[{'id': 'ose-devel', 'name': 'ose-devel',
'baseurl':
'http://buildvm-devops.usersys.redhat.com/puddle/build/OpenShiftEnterprise/3.0/latest/RH7-RHOSE-3.0/$basearch/os',
'enabled': 1, 'gpgcheck': 0}]

# Origin copr repo
#openshift_additional_repos=[{'id': 'openshift-origin-copr', 'name':
'OpenShift Origin COPR', 'baseurl':
'https://copr-be.cloud.fedoraproject.org/results/maxamillion/origin-next/epel-7-$basearch/',
'enabled': 1, 'gpgcheck': 1, gpgkey:
'https://copr-be.cloud.fedoraproject.org/results/maxamillion/origin-next/pubkey.gpg'}]

# host group for masters
[masters]
ose3-master.example.com

# host group for nodes
[nodes]
ose3-node[1:2].example.com
```

The hostnames above should resolve both from the hosts themselves and
the host where ansible is running (if different).

## Running the ansible playbooks
From the atomic-enterprise-ansible checkout run:
```sh
ansible-playbook playbooks/byo/config.yml
```
**Note:** this assumes that the host inventory is /etc/ansible/hosts, if using a different
inventory file use the -i option for ansible-playbook.

## Post-ansible steps
#### Create the default router
On the master host:
```sh
oadm router --create=true \
  --credentials=/etc/openshift/master/openshift-router.kubeconfig \
  --images='docker-buildvm-rhose.usersys.redhat.com:5000/openshift3/ose-${component}:${version}'
```

#### Create the default docker-registry
On the master host:
```sh
oadm registry --create=true \
  --credentials=/etc/openshift/master/openshift-registry.kubeconfig \
  --images='docker-buildvm-rhose.usersys.redhat.com:5000/openshift3/ose-${component}:${version}' \
  --mount-host=/var/lib/openshift/docker-registry
```

## Overriding detected ip addresses and hostnames
Some deployments will require that the user override the detected hostnames
and ip addresses for the hosts. To see what the default values will be you can
run the openshift_facts playbook:
```sh
ansible-playbook playbooks/byo/openshift_facts.yml
```
The output will be similar to:
```
ok: [10.3.9.45] => {
    "result": {
        "ansible_facts": {
            "openshift": {
                "common": {
                    "hostname": "jdetiber-osev3-ansible-005dcfa6-27c6-463d-9b95-ef059579befd.os1.phx2.redhat.com",
                    "ip": "172.16.4.79",
                    "public_hostname": "jdetiber-osev3-ansible-005dcfa6-27c6-463d-9b95-ef059579befd.os1.phx2.redhat.com",
                    "public_ip": "10.3.9.45",
                    "use_openshift_sdn": true
                },
                "provider": {
                  ... <snip> ...
                }
            }
        },
        "changed": false,
        "invocation": {
            "module_args": "",
            "module_name": "openshift_facts"
        }
    }
}
ok: [10.3.9.42] => {
    "result": {
        "ansible_facts": {
            "openshift": {
                "common": {
                    "hostname": "jdetiber-osev3-ansible-c6ae8cdc-ba0b-4a81-bb37-14549893f9d3.os1.phx2.redhat.com",
                    "ip": "172.16.4.75",
                    "public_hostname": "jdetiber-osev3-ansible-c6ae8cdc-ba0b-4a81-bb37-14549893f9d3.os1.phx2.redhat.com",
                    "public_ip": "10.3.9.42",
                    "use_openshift_sdn": true
                },
                "provider": {
                  ...<snip>...
                }
            }
        },
        "changed": false,
        "invocation": {
            "module_args": "",
            "module_name": "openshift_facts"
        }
    }
}
ok: [10.3.9.36] => {
    "result": {
        "ansible_facts": {
            "openshift": {
                "common": {
                    "hostname": "jdetiber-osev3-ansible-bc39a3d3-cdd7-42fe-9c12-9fac9b0ec320.os1.phx2.redhat.com",
                    "ip": "172.16.4.73",
                    "public_hostname": "jdetiber-osev3-ansible-bc39a3d3-cdd7-42fe-9c12-9fac9b0ec320.os1.phx2.redhat.com",
                    "public_ip": "10.3.9.36",
                    "use_openshift_sdn": true
                },
                "provider": {
                    ...<snip>...
                }
            }
        },
        "changed": false,
        "invocation": {
            "module_args": "",
            "module_name": "openshift_facts"
        }
    }
}
```
Now, we want to verify the detected common settings to verify that they are
what we expect them to be (if not, we can override them).

* hostname
  * Should resolve to the internal ip from the instances themselves.
  * openshift_hostname will override.
* ip
  * Should be the internal ip of the instance.
  * openshift_ip will override.
* public hostname
  * Should resolve to the external ip from hosts outside of the cloud
  * provider openshift_public_hostname will override.
* public_ip
  * Should be the externally accessible ip associated with the instance
  * openshift_public_ip will override
* use_openshift_sdn
  * Should be true unless the cloud is GCE.
  * openshift_use_openshift_sdn overrides

To override the the defaults, you can set the variables in your inventory:
```
...snip...
[masters]
ose3-master.example.com openshift_ip=1.1.1.1 openshift_hostname=ose3-master.example.com openshift_public_ip=2.2.2.2 openshift_public_hostname=ose3-master.public.example.com
...snip...
```
