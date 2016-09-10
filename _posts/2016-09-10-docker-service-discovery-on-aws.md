---
layout: post
title: "Docker: Service discovery for docker containers on AWS"
description: "Demonstrate how to do service discovery by using haproxy and route53 on AWS"
category: automation
tags: [ansible, AWS, docker, service discovery, haproxy, route53]
---

These days we see lots of fancy ways to do service discovery with docker. You see examples of people using
etcd, zookeeper, consul and so on. I recently had a project to migrate some solr clusters to docker and I
started looking at some of those tools and how they would fit that our current infrastructure.

After some researching I found that the KISS way of doing service discovery on our environment was to use route53.
The main reasons for this decision were:

- no new moving pieces to our (already complex) architecture
- we already use route53 private zones so no additional costs involved
- simple and yet scalable service
- well tested and proven stable
- easily manipulated with ansible

The idea was simple: every time a docker container was launched, route53 and haproxy had to be updated
simultaneously to point to the new service. Also, I didn't want to bother with static port mappings.

For this article, I'll be using a simple 2 layer-service. Let's say we have "login service" and "publish service".
So there will be 2 docker containers. Let's start with the docker-compose file


## Docker compose

Here's an example:

    version: '2'
    services:
      login_service:
        image: "login_service"
        restart: always
        container_name: loginservice
        hostname: login_service
        domainname: example.com
        volumes:
          - /var/log/loginservice/:/var/log/loginservice
        environment:
          - DOMAIN=login
      publish_service:
        image: "publish_service"
        restart: always
        container_name: publish_service
        volumes:
          - /var/log/publishservice/:/var/log/publishservice
        environment:
          - DOMAIN=publish

Nothing special here. The only thing to note is that we defined a `DOMAIN` environment variable. We're going to
use that environment variable to register the service with route53


## Dump container list

We need to dump the list of containers with all the information about each container we can possibly have. This
small python snipped does the trick:

```
def get_containers(cli):
    container_ids = [ c['Id'] for c in cli.containers() ]
    containers = [ cli.inspect_container(cid) for cid in container_ids ]

    for container in containers:
        env_list = container['Config']['Env']
        env_dict = {}
        for e in env_list:
            (k,v) = e.split('=')
            env_dict[k] = v
        container['Config']['Env'] = env_dict

    return containers

...

def dump_yml(args):
    import yaml
    cli = Client()
    containers = get_containers(cli)
    yaml.safe_dump({ 'containers': containers }, sys.stdout, default_flow_style=False)

```

The full script is available at: https://gist.github.com/filipenf/9ce4b94a06b2d6eb99d05f293e69dbe1

Save this file into `/usr/bin/docker-dump.py` and make it executable


## haproxy configuration template

Ansible have jinja2 templating built-in. This makes easy for us to define a haproxy.cfg template that
will build the final haproxy configuration from the list of containers:

```
{%raw%}
global
    daemon
    maxconn 4096
defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
frontend http
    bind *:80
{%- for container in containers %}
    acl acl_{{ container.Config.Env.DOMAIN }} hdr(host) -m beg -i {{ container.Config.Env.DOMAIN }}
    use_backend {{ container.Config.Env.DOMAIN }}_backend if acl_{{ container.Config.Env.DOMAIN }}
{% endfor -%}

{% for container in containers %}
backend {{ container.Config.Env.DOMAIN }}_backend
    option http-tunnel
    option forwardfor
{%- set name = container.Config.Env.DOMAIN %}
{%- set ip = container.NetworkSettings.Networks[container.NetworkSettings.Networks.keys()[0]].IPAddress %}
{%- set port = container.NetworkSettings.Ports.keys()[0].split('/')[0] %}
    server {{ name }} {{ ip }}:{{ port }}
{% endfor -%}
{% endraw %}
```

Here we're using the `DOMAIN` environment variable that we got from the container to do the magic.
Of course we may have containers that do not have that variable defined. We'll filter those on ansible.

Put this template into some place ansible will be able to read later (i.e `/etc/haproxy_template.j2`)

## Ansible playbook to update haproxy and route53

Here's the playbook we'll use to update both route53 and haproxy. Let's call it update-services.yml:

```
{%raw%}
---
- hosts: localhost
  gather_facts: yes

  vars:
    domain_name: example.com

  pre_tasks:
    - action: ec2_facts

  handlers:
    - name: restart haproxy
      service: name=haproxy state=restarted

  tasks:
    - command: /usr/bin/docker-dump.py dump-yml
      register: dump

    - set_fact: containers="{{ dump.stdout | from_yaml | json_query('containers[?Config.Env.DOMAIN]') }}"

    - name: update route53 from containers
      route53:
        command: create
        overwrite: yes
        private_zone: yes
        zone: '{{ domain_name }}'
        record: '{{ container.Config.Env.DOMAIN }}.{{ domain_name }}'
        value: "{{ ansible_ec2_local_ipv4 }}"
        ttl: 300
        type: A
      with_items: '{{ containers }}'
      loop_control:
        loop_var: container

    - name: generate haproxy configuration
      template: src=/etc/haproxy_template.j2 dest=/etc/haproxy/haproxy.cfg
      notify: restart haproxy

{%endraw%}
```

This playbook is meant to be executed on the docker host. Now you just need to execute the playbook:

`ansible-playbook update-services.yml`

and the new services will be registered with haproxy. If something changed, haproxy will restart
automatically. Also, the domain names will be registered/update on route53.

## Note: Route53 permissions

Your instance's IAM profile will need to have permission to update route53. A policy like:

```
{
   "Statement":[
      {
         "Action":[
            "route53:ChangeResourceRecordSets",
            "route53:GetHostedZone",
            "route53:ListResourceRecordSets"
         ],
         "Effect":"Allow",
         "Resource":[
            "arn:aws:route53:::hostedzone/<Your zone ID>"
         ]
         ....
},

```

should do the trick. Take a look at: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/auth-and-access-control.html
for more information

## Conclusion

This is just a small example of what you can do. Of course you may end up with much more complex situations where the same
haproxy frontend should map to different backends depending on which port it came from and so on. Anyway, the above 'framework'
allows you to do that with a bit more jinja templating on the haproxy template.

I hope the ideas shown here can help other guys with similar situations to simplify their deployments.
