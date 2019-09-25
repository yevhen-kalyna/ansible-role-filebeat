# Filebeat
=========

Created for direct log sending to Elasticsearch AWS. But easy can be used for writing to Logstash and on-demand Elasticsearch server.

Have nice feature with config like vars in Ansible, it's mean you can write filebeat config "as is" and it will used for Filebeat, easy for users who don't wont lear a specific of the role.

# Testing
-------
Some images don't have CA for requests and will fail. # TODO: Find images with CA installed.
Need to have installed and configured Molecule.

```bash
molecule converge
```

Create docker containers and deploy role.

```bash
molecule destroy
```

Remove containers

```bash
molecule create
```

Create and prepare containers

```bash
molecule lint
```

Run yamllint.

```bash
molecule converge -- --tags config
```

Redeploy config only and restart service.

# Requirements
------------

Tested on next OS:
 * RHEL:
   * CentOS 6;
   * CentOS 7;
   * Fedora 30;
 * Debian:
   * Debian 9;
   * Debian 10;
   * Ubuntu 16.04;
   * Ubuntu 18.04;

Do not use filebeat.module when your Filebeat output is Elasticsearch, because a lot of modules use geoip and AWS doesn't support its plugin.

# Role Variables
--------------

FYI: In folder ansible-role-filebeat/files saved all configs and variables for Filebeat 5.0,6.0,6.7,7.0,7.3 versions. User it for configuration filebeat.

Version of repository.

```yaml
es_major_version: "7.x"
```

Use Open Distro repository? Only for versions 7.x +.

```yaml
es_is_oss: true
```

Version of filebeat

```yaml
es_version: "7.3.0"
```

Need to create filebeat configuration file?

```yaml
filebeat_create_config: "true"
```

Need to enable filebeat for starting on boot?

```yaml
filebeat_enabled: "yes"
```

Elastic package repository key.

```yaml
filebeat_repo_key: 'https://artifacts.elastic.co/GPG-KEY-elasticsearch'
```

Status of filebeat service after deploy will be done.

```yaml
filebeat_run_state: started
```

Separated by coma elasticsearch hosts.

```yaml
filebeat_output_elasticsearch: "localhost:9200"
```

Use elasticsearch like output host?

```yaml
filebeat_output_elasticsearch_enabled: true
```

Separated by coma logstash hosts.

```yaml
filebeat_output_logstash: "localhost:5044","1.2.3.4:5044"
```

Use logstash like output host?

```yaml
filebeat_output_logstash_enabled: false
```

Use certificate for securing connection between filebeat and output?

```yaml
filebeat_ssl_enabled: false
```

Can be connection insecure?

```yaml
filebeat_ssl_insecure: "false"
```

Location of the crt and key on remote machine.

```yaml
filebeat_ssl_dir: /etc/ssl/certs
```

CRT for secure connection between output and filebeat. Must be in one file layout with playbook.

```yaml
filebeat_ssl_certificate_file: "mydomain.crt"
```

CA for secure connection between output and filebeat. Must be in one file layout with playbook. If you create CA like CRT bundle, fill its var empty.

```yaml
filebeat_ssl_certificate_authorities_file: "mydomain.pem"
```

Key for your CRT for secure connection between output and filebeat. Must be in one file layout with playbook.

```yaml
filebeat_ssl_key_file: "mydomain.key"
```

Logs will be sended to output with this index. By default index will be changed according to input's field with name "type", if not have "type" field, index will contain "other".

```yaml
filebeat_index: "filebeat-%{[fields.type]:other}-%{+yyyy.MM}"
```

Put config for your Filebeat "as is" in filebeat_config_content variable.

```yaml
filebeat_config_content: |
  filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/*.log
    fields:
      type: "logs"
  output.elasticsearch:
    enabled: true
    hosts: ["{{ filebeat_output_elasticsearch }}"]
    index: "{{ filebeat_index }}"
```

# Dependencies
------------

None.

# Example Playbook
----------------

Including an example of how to use role:

### Minimal with default configs for AWS:ES:

```yaml
---
- hosts: all
  roles:
    - role: yevhen_kalyna.filebeat
  vars:
    filebeat_output_elasticsearch: "https://mydomain.eu-west-1.aws.com:443"
```

### For AWS:ES and docker logging:

```yaml
---
- hosts: all
  roles:
    - role: yevhen_kalyna.filebeat
  vars:
    filebeat_output_elasticsearch: "https://mydomain.us-west-2.es.amazonaws.com:443"
    filebeat_output_elasticsearch_enabled: true
    filebeat_config_content: |
      name: "{{ inventory_hostname }}"

      filebeat.inputs:
      - type: container
        enabled: true
        fields:
            type: "docker"
        containers:
          ids:
            - '*'
        paths:
          - '/var/lib/docker/containers/*/*.log'
      processors:
      - add_cloud_metadata: ~
      - add_docker_metadata: ~
      - add_host_metadata:
          netinfo.enabled: false

      filebeat.autodiscover:
        providers:
          - type: docker
            templates:
              - condition.or:
                  - contains.docker.container.name: "nginx"
                config:
                  - type: docker
                    containers.ids:
                      - ${data.docker.container.id}

      output.elasticsearch:
        enabled: true
        hosts: ["{{ filebeat_output_elasticsearch }}"]
        compression_level: 3
        index: "{{ filebeat_index }}"

      setup.template.enabled: false
      setup.template.name: "{{ filebeat_index }}"
      setup.template.pattern: "filebeat-%{[fields.type]:other}-*"
      setup.ilm.enabled: false

      logging.level: info
      logging.to_files: true
      logging.files:
        path: /var/log/filebeat
        name: filebeat.log
        rotateeverybytes: 10485760 # = 10MB
        keepfiles: 7
        permissions: 0600
```

### For On-demand Elasticsearch server with geoip plugin for logging basic thinks and docker

```yaml
- hosts: all
  roles:
    - role: ansible-role-filebeat
  vars:
    filebeat_output_elasticsearch: "https://search-temp-test-u2ndgsvn3x7khcooari4enmocu.us-west-2.es.amazonaws.com:443"
    filebeat_output_elasticsearch_enabled: true
    filebeat_config_content: |
      name: "{{ inventory_hostname }}"

      filebeat.modules:
      - module: system
        syslog:
          enabled: true
          fields:
            type: "syslog"
        auth:
          enabled: true
          fields:
            type: "auth"

      - module: auditd
        log:
          enabled: true
          fields:
            type: "auditd"

      filebeat.inputs:
      - type: container
        enabled: true
        fields:
            type: "docker"
        containers:
          ids:
            - '*'
        paths:
          - '/var/lib/docker/containers/*/*.log'
      processors:
      - add_cloud_metadata: ~
      - add_docker_metadata: ~
      - add_host_metadata:
          netinfo.enabled: false
      - add_process_metadata:
          match_pids: ["system.process.ppid"]
          target: system.process.parent

      filebeat.autodiscover:
        providers:
          - type: docker
            templates:
              - condition.or:
                  - contains.docker.container.name: "nginx"
                config:
                  - type: docker
                    containers.ids:
                      - ${data.docker.container.id}

      output.elasticsearch:
        enabled: true
        hosts: ["{{ filebeat_output_elasticsearch }}"]
        compression_level: 3
        index: "{{ filebeat_index }}"

      setup.template.enabled: false
      setup.template.name: "{{ filebeat_index }}"
      setup.template.pattern: "filebeat-{[fields.type]:other}-*"
      setup.ilm.enabled: false

      logging.level: info
      logging.to_files: true
      logging.files:
        path: /var/log/filebeat
        name: filebeat.log
        rotateeverybytes: 10485760 # = 10MB
        keepfiles: 7
        permissions: 0600
```

License
-------

BSD

Author Information
------------------

Yevhen Kalyna

https://www.linkedin.com/in/yevhen-kalyna/
https://gitlab.com/yevhen.kalyna
https://github.com/yevhen-kalyna