
### Ansible tip of the day: Using with_nested to merge a list with a dict

I make heavy use of Ansible for AWS infrastructure provisioning. From creating VPCs, subnets, EC2 instances, security groups and so on.

Because we have some standard central point where we define per-environment networks, I recently got stuck with having to convert a list of subnets (just strings) to a list of dicts to be used by the `ec2_group` module. My list of subnets was something like:

```
internal_networks:
  - 10.100.32.0/20
  - 10.100.128.0/20
  - 10.100.144.0/20
  - 10.50.128.0/20
  - 10.50.144.0/20
```

While the `ec2_group` expects its rules to be something like:

```
      - proto: tcp
        from_port: 80
        to_port: 80
        cidr_ip: 0.0.0.0/0
```

So if it was a programming language, it was just a simple `for` loop to achieve that. In the playbook world though, things are a little bit trickier.

To workaround that, I first defined the dict *skeleton* like:

```
general_rules:
  - proto: all
    from_port: -1
    to_port: -1
```

Of course, I can have any number of rules inside that list. Each one will be combined with each item of my `internal_networks` list.

Now, to merge the `general_rules` and `internal_networks` data structures together I used **set_fact**:

```
- set_fact:
  args:
    rule:
      proto: "{{ item.1.proto }}"
      from_port: "{{ item.1.from_port }}"
      to_port: "{{ item.1.to_port }}"
      cidr_ip: "{{ item.0 }}"
  with_nested:
    - "{{ internal_networks }}"
    - "{{ general_rules }}"
  register: merged_rules

- name: "Merge networks with rules"
  set_fact: merged_rules="{{merged_rules.results | map(attribute='ansible_facts.rule') | list }}"
```

The final data structure should look like:

```
       "all_rules": [
            {
                "cidr_ip": "10.100.32.0/20",
                "from_port": "-1",
                "proto": "all",
                "to_port": "-1"
            },
            {
                "cidr_ip": "10.100.128.0/20",
                "from_port": "-1",
                "proto": "all",
                "to_port": "-1"
            },
```

You can find the complete example [here](https://gist.github.com/filipenf/4869cf597c264031f529). Hope it helps :-)