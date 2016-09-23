# Heat with Ansible
Heat stack to create simple server using Ansible to install and start nginx

```yaml
heat_template_version: 2014-10-16

description: |
  Deploy Nginx server with Ansible
```
Parameters used by template - use -P option to stack-create to set
```yaml
parameters:
  image_id:
    type: string
    default: 9ab886e6-c75c-4279-9418-dd1ba7876335
  network_id:
    type: string
    default: 0c9b2542-3d24-48b6-8de8-8c4f8a187ccc
  server_name:
    type: string
    default: centos_w_nginx
  key_name:
    type: string
    default: mykey
```
Resources to create as part of stack - first resource is an Ansible playbook that gets passed to the ansbible hooks 
when the server is created
```yaml
resources:
  nginx_config:
    type: OS::Heat::SoftwareConfig
    properties:
      group: ansible
      config: |
        ---
        - name: Install and run Nginx
          connection: local
          hosts: localhost
          tasks:
           - name: Install Nginx
             yum: name=nginx state=installed 
             notify:
              - Start Nginx
          handlers:
           - name: Start Nginx
             service: name=nginx state=started
```
SoftwareDeployment resource connects the SoftwareConfig resource to the server

```yaml
  deploy_nginx:
    type: OS::Heat::SoftwareDeployment
    properties:
      signal_transport: TEMP_URL_SIGNAL
      config: 
        get_resource: nginx_config
      server: { get_resource: server }
```
Create a floating IP for external access to the web server
```yaml
  floatingip:
    type: OS::Nova::FloatingIP
    properties:
      pool: nova
```
Associate the floating IP with the server
```yaml
  floatingipassoc:
    type: OS::Nova::FloatingIPAssociation
    properties:
      floating_ip: { get_resource: floatingip }
      server_id: { get_resource: server }
```
Create a simple server using the image with the Ansible hooks
```yaml
  server:
    type: OS::Nova::Server
    properties:
      image: { get_param: image_id }
      flavor: m1.small
      networks:
        - network: { get_param: network_id }
      key_name: { get_param: key_name }
      software_config_transport: POLL_TEMP_URL
      user_data_format: SOFTWARE_CONFIG
      name: { get_param: server_name }
      security_groups: [ default, { get_resource: security_group } ]
```
Add a security group to make sure we can http, ssh and ping the instance

```yaml
  security_group:
    type: OS::Neutron::SecurityGroup
    properties:
      description: Enable http, ssh and icmp access to the server
      name: 'SecurityGroup'
      rules: [
        { remote_group_id: null,
        direction: ingress,
        remote_ip_prefix: 0.0.0.0/0,
        protocol: tcp,
        port_range_max: 22,
        port_range_min: 22,
        ethertype: IPv4 },
        { direction: ingress,
        remote_ip_prefix: 0.0.0.0/0,
        protocol: icmp,
        ethertype: IPv4 },
        { direction: ingress,
        remote_ip_prefix: 0.0.0.0/0,
        protocol: tcp,
        port_range_max: 80,
        port_range_min: 80,
        ethertype: IPv4 } ]
```