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
```
Upload image to glance
```
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
  
Create stack with appropriate parameters (I have defaults in template except for new image_id)
```
heat stack-create -f nginx_server.yml -P image_id=e4ce978a-5650-4752-8c44-7da160f32117 test-nginx
+--------------------------------------+------------+--------------------+---------------------+--------------+
| id                                   | stack_name | stack_status       | creation_time       | updated_time |
+--------------------------------------+------------+--------------------+---------------------+--------------+
| d9b24d6f-d81c-4c25-83db-ea2f1a6cbe19 | test-nginx | CREATE_IN_PROGRESS | 2016-09-22T20:23:37 | None         |
+--------------------------------------+------------+--------------------+---------------------+--------------+
```
Verify resources created

```
heat resource-list test-nginx 
+-----------------+--------------------------------------+---------------------------------+-----------------+---------------------+
| resource_name   | physical_resource_id                 | resource_type                   | resource_status | updated_time        |
+-----------------+--------------------------------------+---------------------------------+-----------------+---------------------+
| deploy_nginx    | 01623c2a-d146-4326-814d-50ec26953a48 | OS::Heat::SoftwareDeployment    | CREATE_COMPLETE | 2016-09-22T20:23:38 |
| floatingip      | c54228b0-4c9b-45f7-be97-3d6817534dc9 | OS::Nova::FloatingIP            | CREATE_COMPLETE | 2016-09-22T20:23:38 |
| floatingipassoc | 60                                   | OS::Nova::FloatingIPAssociation | CREATE_COMPLETE | 2016-09-22T20:23:38 |
| nginx_config    | 20c39aa5-5064-456d-8d6d-285aed6f33c0 | OS::Heat::SoftwareConfig        | CREATE_COMPLETE | 2016-09-22T20:23:38 |
| server          | c564d0a0-6bcb-41c5-a01f-c4aa8ad711db | OS::Nova::Server                | CREATE_COMPLETE | 2016-09-22T20:23:38 |
+-----------------+--------------------------------------+---------------------------------+-----------------+---------------------+
```
Get floating IP for VM
```
nova list
+--------------------------------------+----------------+---------+------------+-------------+---------------------------------+
| ID                                   | Name           | Status  | Task State | Power State | Networks                        |
+--------------------------------------+----------------+---------+------------+-------------+---------------------------------+
| c564d0a0-6bcb-41c5-a01f-c4aa8ad711db | centos_w_nginx | ACTIVE  | -          | Running     | private=172.0.2.74, 192.0.2.144 |
+--------------------------------------+----------------+---------+------------+-------------+---------------------------------+
```
Test nginx

```
curl 192.0.2.144
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">

<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
    <head>
        <title>Test Page for the Nginx HTTP Server on Fedora</title>
...
```