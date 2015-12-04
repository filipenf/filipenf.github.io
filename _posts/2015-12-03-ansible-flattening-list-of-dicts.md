---
layout: post
title: "Ansible: Flattening a list of dicts to a single dict"
description: ""
category:
tags: []
---

Ansible is a great tool for automation. It has lots of modules that makes our lives easier. As a heavy-user of ansible for infrastructure provisioning on AWS, I sometimes get frustrated to have to deal with different data structures used by modules.

One example I had to deal with recently was the tags attribute of ec2_asg and ec2_tag. You cannot just pass the same value for both of them because ec2_asg tags attribute expects a list of dicts, while ec2_tag expects just a dict. That difference is because ec2_asg's dicts have an optional attribute called *propagate_at_launch*.

So, the big thing here is that I have one *project definition* file with variables that are used all over. So in that project yml, I put my global tags like:

    project_tags:
      - Environment: "{{ project_env }}"
      - ProjectName: "{{ project_name }}"


As you can see, this is a list of dicts, and its already suitable to use with ec2_asg module (since the optional *propagate_at_launch* defaults to yes ). Now, how I make that structure suitable to use with ec2_tag?

I've searched for built-in filters that could do the job, but unfortunately didn't find. Turns out that writing your own filter plugin is really simple, and that filter can be created inside your project repository which makes easier to share with your teammates.

## Create your filter plugin

In your main ansible roles directory structure, create a *filter_plugins* directory. And inside that directory, create a flatten_list.py with the following code:

{% gist 7bcaa45cce3bba5b714b %}

So your ansible's structure will be something like:

    ➜  setup git:(mcp) ✗ tree
    .
    ├── README.md
    ├── filter_plugins
    │   └── flatten_list.py
    ├── group_vars
    │   └── all
    ├── launch-hubot.yml
    ├── launch-....yml
    ├── lookup_plugins
    │   └── find_subnet.py
    ├── projects
    │   ├── hubot.yml
    │   ├── templates
    │   │   └── project-template.yml
    ├── roles
    │   ├── autoscaling
    │   │   └── tasks
    ....
    │   ├── volumes
    │   │   └── tasks

Now, you just need to pass that list of dicts to the flatten_list filter and voilà!

  - name: "Tag volumes"
    local_action:
      module: ec2_tag
      region: "{{ project_region }}"
      resource: "{{ item.volume.id }}"
      tags:
        "{{ project_tags | flatten_list }}"
    with_items:
      - "{{ volumes.results }}"



