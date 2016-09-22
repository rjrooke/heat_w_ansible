# OpenStack Heat with [Ansible](https://www.ansible.com/)

## Description

This is a basic Heat template to test using Ansible 
with Heat templates.  Similar to procedure outlined here:
https://developer.rackspace.com/docs/user-guides/orchestration/ansible/using-ansible-with-heat/


## Setup

### Disk image

Need a guest image with Ansible hooks and I don't think the default 
images come with them.  Use procedure similar to this reference:
 https://github.com/openstack/heat-templates/tree/master/hot/software-config/elements
I've modified it to use centos7 - this requires a couple extra elements and I dropped some.

```
git clone https://git.openstack.org/openstack/diskimage-builder.git
git clone https://git.openstack.org/openstack/tripleo-image-elements.git
git clone https://git.openstack.org/openstack/heat-templates.git
git clone https://git.openstack.org/openstack/dib-utils.git

export PATH="${PWD}/dib-utils/bin:$PATH"
export ELEMENTS_PATH=tripleo-image-elements/elements:heat-templates/hot/software-config/elements

diskimage-builder/bin/disk-image-create vm   centos7 selinux-permissive \
    os-collect-config   \
    os-refresh-config   \
    os-apply-config   \
    heat-config   \
    heat-config-ansible   \
    heat-config-cfn-init    \
    heat-config-puppet     \
    heat-config-script \
    ansible \
    epel  \
    -o centos7-software-config.qcow2
glance image-create --file centos7-software-config.qcow2 --name centos7-software-config --progress --visibility public --container-format bare --disk-format qcow2
[=============================>] 100%
+------------------+-----------------------------------------------------------------+
| Property         | Value                                                           |
+------------------+-----------------------------------------------------------------+
| checksum         | 7db88f2fc31cdf2f0238c33614c99796                                |
| container_format | bare                                                            |
| created_at       | 2016-09-22T20:22:10Z                                            |
| direct_url       | swift+config://ref1/glance/e4ce978a-5650-4752-8c44-7da160f32117 |
| disk_format      | qcow2                                                           |
| id               | e4ce978a-5650-4752-8c44-7da160f32117                            |
| min_disk         | 0                                                               |
| min_ram          | 0                                                               |
| name             | centos7-software-config                                         |
| owner            | 9470f8dcd1ee4cb0900fba390d45703a                                |
| protected        | False                                                           |
| size             | 669773824                                                       |
| status           | active                                                          |
| tags             | []                                                              |
| updated_at       | 2016-09-22T20:22:22Z                                            |
| virtual_size     | None                                                            |
| visibility       | public                                                          |
+------------------+-----------------------------------------------------------------+

```

### Heat template

This template allocates a floating IP, so will have some available

Identify values for required parameters:

- image_id: ID for glance image created earlier
- network_id: Network ID to use for VM - should be a private network as this template creates a floating IP
- server_name: Name to give to the server
- key_name: name of a nova keypair
  
