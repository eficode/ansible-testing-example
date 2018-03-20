# Testing ansible roles

It is a good practice to place roles into a public or private galaxy and automatically test that they work after every change.

## Prerequisites
- [Docker](https://www.docker.com/get-docker)
- [Python](https://www.python.org/downloads/)
- [Virtualenv](https://virtualenv.pypa.io/en/stable/installation/)

## Preparations
Create and activate virtualenv
```
virtualenv venv
source venv/bin/activate
```
Install Ansible and Molecule
```
pip install -r requirements.txt
```
## Getting started with molecule

### Init role
```
molecule init role --role-name complicated-nginx
cd complicated-nginx
```

### Add a complicated template for nginx frontpage
```
mkdir templates
cat <<'EOF' > templates/index.html.j2
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.

<strong>Also please mind that here's some overly complicated stuff that we might see for example in configuration files sometimes:</strong>

{% for c in raw_data if mapping[c] in "deadbeef" %}{{ c }}{% endfor %}

</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
EOF
```

### Add some default variables
```
cat <<'EOF' > defaults/main.yml
---
raw_data: "eulmfkillcloldkue"
mapping:
  q: o
  c: d
  e: d
  o: b
  i: a
  d: e
  p: u
  r: k
  a: l
  m: o
  f: e
  l: u
  k: z
  u: w
EOF
```

### Tasks for generating frontpage and running nginx
```
cat <<'EOF' > tasks/main.yml
---
- name: generate index.html
  template:
    src: index.html.j2
    dest: /usr/share/nginx/html/index.html

- name: start nginx
  shell: nginx
  tags: skip_ansible_lint
  register: start_nginx
  failed_when: "start_nginx.rc != 0 and 'Address in use' not in start_nginx.stderr"
  changed_when: false
EOF
```

### Edit molecule file
```
cat <<'EOF' > molecule/default/molecule.yml
---
dependency:
  name: galaxy
driver:
  name: docker
lint:
  name: yamllint
platforms:
  - name: instance
    image: nginx:alpine
    published_ports:
      - "0.0.0.0:80:80/tcp"
provisioner:
  name: ansible
  lint:
    name: ansible-lint
scenario:
  name: default
verifier:
  name: testinfra
  lint:
    name: flake8
EOF
```

### Converge
**NOTE:** before converging, please look at defaults and the index template and try to figure out what it will result into. This is just an example but there are many cases where infra code is complicated and testing needs to be in place. For example iptable rules could be generated in similar manner.
```
molecule converge
```

### See result
Open [localhost](http://localhost). Did you guess the answer?

### Make tests
```
cat <<'EOF' > molecule/default/tests/test_default.py
import os

import testinfra.utils.ansible_runner

testinfra_hosts = testinfra.utils.ansible_runner.AnsibleRunner(
    os.environ['MOLECULE_INVENTORY_FILE']).get_hosts('all')


def test_hosts_file(host):
    f = host.file('/usr/share/nginx/html/index.html')
    assert "lollerskates" in f.content
EOF
```
Can you spot the error in provided test case?
