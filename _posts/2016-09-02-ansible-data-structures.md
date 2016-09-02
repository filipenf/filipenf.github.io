---
layout: post
title: "Ansible: Data structure handling and transformation"
description: "This post will show how you can make use of the json_query filter to query your data"
category: automation
tags: [ansible, data structure, json_query, jmespath]
---

From time to time I had some dificulties with handling data structures with ansible. Things like converting
a list of dicts to a list of values ( inside a dict ) or even filtering a list aren't always trivial.

Because of that, I've written a small filter plugin called `json_query`, that allows you to do some cool
transformations with ansible. The filter uses jmespath ( http://jmespath.org/ ) a query language for
json data. The plugin is already merged to `devel` branch and will be released on 2.2 version of ansible.

Essentially, you feed the filter with a data structure and a query string. The filter will then return
the data you have queried. Some example of jmespath queries:

- `[8:10]` - returns the elements 8, 9 and 10
- `[:5]` - returns the range from the first to 5th element on a list
- `[::2]` - returns every other element on a list
- `users[*].name` - returns a list containing all user names
- `users[0].name` - only the name of the first user
- `reservations[*].instances[*].instance_id` - prints all the instance ids

More examples can be found at: http://jmespath.org/tutorial.html. They have even an expression tester that
helps you to validate your jmespath queries.

## Examples

### 1. Filtering users by admin level

Let's say you have a list of users like this:

    users:
      - name: admin_user
        admin: yes
        uid: 1000
      - name: foo
        admin: no
        uid: 1001
      - name: bar
        uid: 1002

and wants to get a list of the users which have the `admin` flag set to true. This is as simple as:

{%raw%}
    - debug: msg="{{ users | json_query(\"[?admin].name\") }}"

{%endraw%}

it will print something like:

    ok: [localhost] => {
        "admins": [
            "admin_user"
        ]
    }

### 2. Tag a list of EBS volumes

Suppose you need to make sure all of your instance's volumes are tagged correctly. You'll need to list
your volumes and than tag each one of them. Here's a simple example showing how to do it:

{%raw%}

    - name: List instance volumes
      ec2_vol:
        region: "us-east-1"
        instance: "{{ instance_id }}"
        state: list
      register: instance_volumes

    - name: Tag volumes
      ec2_tag:
        region: "us-east-1"
        resource: "{{ item }}"
        tags:
          Name: "{{ instance_id }} volume {{ item }}"
		  Owner: "bla"
      with_items: "{{ instance_volumes | json_query(\"volumes[*].id\"}}"

{%endraw%}


### 3. Parse data from web services

Here's a simple example of how powerful this feature can be. Imagine that you have some external source
of configuration, like a web service that will return a list of backend servers to connect to.

Just to exemplify, here's how you would do it. This example fetches the list of currently open tickets
on github then prints only the ones that are labeled `in progress`:

{% raw %}

    - uri:
        url: https://api.github.com/repos/ansible/ansible/issues?state=open
        body_format: json
      register: uri_result
    - debug: msg="{{ uri_result.json | json_query(\"[?labels[?name=='in progress']][title,id]\") }}"

{% endraw %}





