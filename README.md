pulp
====

Ansible role which installs and confiures the Pulp Project. The role was tested
in single host installation but it should be possible to spread the Pulp service
across multiple hosts and even use different messaging tool than Qpid (e.g.
RabbitMQ).

The configuration of the role is done in such way that it should not be necessary
to change the role for any kind of configuration. All can be done either by
changing role parameters or by declaring completely new configuration as a
variable. That makes this role absolutely universal. See the examples below for
more details.

Please report any issues or send PR.


Examples
--------

This role was tested with single-host setup only and this is the configuration
which worked for me:

```
---

# Example of a single host installation
- hosts: myhost
  vars:
    # MongoDB configuration
    mongodb_net_bindIp: 127.0.0.1
    mongodb_net_wireObjectCheck: false
    mongodb_net_unixDomainSocket_enabled: true
    mongodb_processManagement_fork: true
    mongodb_systemLog_logAppend: true
    mongodb_systemLog_timeStampFormat: iso8601-utc
    # Qpid configuration
    qpid_cpp_server_qpidd_config:
      auth: 'no'
    # Pulp configuration
    pulp_install_server: true
    pulp_install_admin: true
    pulp_install_consumer: true
    pulp_run_celerybeat: true
    pulp_run_resource_manager: true
  roles:
    - mongodb
    - qpid_cpp_server
    - pulp
```

After everything is installed, you can use the following commands to create a clone of a repo:

```
pulp-admin login -u admin -p admin
pulp-admin rpm repo create --repo-id=base-el6-64-extras-20150103 --display-name=base-el6-64-extras-20150103 --relative-url=base/centos6/x86_64/extras/20150103 --serve-http=true --feed=http://mirror.centos.org/centos/6/extras/x86_64/
pulp-admin rpm repo sync run --repo-id=base-el6-64-extras-20150103
pulp-admin rpm repo list
```

The repo should then be accessible via URL `http://myhost/pulp/repos/`.

It should also be possible to use it in multi-host setup, although I did not test
it. The setup should look something like this:

```
---

# Mongo DB server
- hosts: pulp-mongodb
  vars:
    mongodb_net_bindIp: 0.0.0.0
    mongodb_net_wireObjectCheck: false
    mongodb_net_unixDomainSocket_enabled: true
    mongodb_processManagement_fork: true
    mongodb_systemLog_logAppend: true
    mongodb_systemLog_timeStampFormat: iso8601-utc
  roles:
    - mongodb

# Messaging server
- hosts: pulp-messaging
  vars:
    # Configure Qpid for multi-host setup
    qpid_cpp_server_qpidd_config:
      ...
  roles:
    # We can use the qpid role directly
    - qpid_cpp_server

# First Pulp server
- hosts: pulp-server1
  vars:
    pulp_install_server: true
    # Only the first Pulp server will run these two services
    pulp_run_celerybeat: true
    pulp_run_resource_manager: true
    # Configure Pulp server for multi-host setup
    pulp_server_config:
      ...
    # Configure repo_auth
    pulp_server_repo_auth_config:
      ...
  roles:
    - pulp

# Second Pulp server
- hosts: pulp-server2
  vars:
    pulp_install_server: true
    # Configure Pulp server for multi-host setup
    pulp_server_config:
      ...
    # Configure repo_auth
    pulp_server_repo_auth_config:
      ...
  roles:
    - pulp

# Pulp admin
- hosts: pulp-admin
  vars:
    pulp_install_admin: true
    # Configure Pulp admin for multi-host setup
    pulp_admin_config:
      ...
  roles:
    - pulp

# Pulp customer
- hosts: pulp-consumer
  vars:
    pulp_install_consumer: true
    # Configure Pulp consumer for multi-host setup
    pulp_consumer_config:
      ...
  roles:
    - pulp
```

It should also be possible to use RabbitMQ instead of the default Qpid messaging
system. Again, I did not test it but the setup should look something like this:

```
---

# Example how to setup Pulp with RabbitMQ on a single host
- hosts: myhost
  vars:
    # MongoDB configuration
    mongodb_net_bindIp: 127.0.0.1
    mongodb_net_wireObjectCheck: false
    mongodb_net_unixDomainSocket_enabled: true
    mongodb_processManagement_fork: true
    mongodb_systemLog_logAppend: true
    mongodb_systemLog_timeStampFormat: iso8601-utc
    # Get some rabbitmq role and configure it (see https://github.com/jtyr/ansible-rabbitmq)
      ...
    # Pulp configuration
    pulp_install_server: true
    pulp_install_admin: true
    pulp_install_consumer: true
    pulp_run_celerybeat: true
    pulp_run_resource_manager: true
    # The server and customer YUM groups are different for RabbitMQ
    pulp_server_pkgs:
      - "@pulp-server"
      - python-gofer-amqplib
    pulp_consumer_pkgs:
      - "@pulp-consumer"
      - python-gofer-amqplib
    # You will probably need to change some configuration
    pulp_server_config:
      messaging:
        transport: rabbitmq
        broker_url: amqp://guest:guest@localhost/foo
        ...
      ...
    pulp_consumer_config:
      messaging:
        transport: rabbitmq
        broker_url: amqp://guest:guest@localhost/foo
        ...
      ...
  roles:
    - mongodb
    - rabbitmq
    - pulp
```


Role variables
--------------

List of variables used by the role:

```
# Whether to install Pulp server
pulp_install_server: false

# Whether to install Pulp admin client
pulp_install_admin: false

# Whether to install Pulp consumer
pulp_install_consumer: false

# Whether to run pulp_celerybeat service
# (should run only on one Pulp server)
pulp_run_celerybeat: false

# Whether to run pulp_resource_manager service
# (should run only on one Pulp server)
pulp_run_resource_manager: false

# Web service name
pulp_web_service: httpd

# Web service user
pulp_web_service_user: apache

# YUM repo URL
pulp_yumrepo_url: https://repos.fedorapeople.org/repos/pulp/pulp/stable/latest/$releasever/$basearch/

# Additionl YUM repo params
pulp_yumrepo_params: {}

# Default YUM group for Pulp server
pulp_server_pkgs:
  - "@pulp-server-qpid"
  # For RabbitMQ use this instead:
  #- "@pulp-server"
  #- python-gofer-amqplib

# Default YUM group for Pulp consumer
pulp_consumer_pkgs:
  - "@pulp-consumer-qpid"
  # For RabbitMQ use these instead:
  #- "@pulp-consumer"
  #- python-gofer-amqplib

# Whether to enable or disable SSL verification
pulp_verify_ssl: false

# Default FQDN of the host (only for single machine installation)
pulp_host: localhost.localdomain

# Default Qpid daemon configuration
pulp_qpidd_config:
  auth: 'no'


# Default content of the Pulp Server sections
pulp_server_config_database: {}
pulp_server_config_server: {}
pulp_server_config_authentication: {}
pulp_server_config_security: {}
pulp_server_config_consumer_history: {}
pulp_server_config_data_reaping: {}
pulp_server_config_oauth: {}
pulp_server_config_messaging: {}
pulp_server_config_tasks: {}
pulp_server_config_email: {}

# Default Pulp Server config sections
pulp_server_config__default:
  database: "{{ pulp_server_config_database }}"
  server: "{{ pulp_server_config_server }}"
  authentication: "{{ pulp_server_config_authentication }}"
  security: "{{ pulp_server_config_security }}"
  consumer_history: "{{ pulp_server_config_consumer_history }}"
  data_reaping: "{{ pulp_server_config_data_reaping }}"
  oauth: "{{ pulp_server_config_oauth }}"
  messaging: "{{ pulp_server_config_messaging }}"
  tasks: "{{ pulp_server_config_tasks }}"
  email: "{{ pulp_server_config_email }}"

# Custom Pulp Server config sections
pulp_server_config__custom: {}

# Final Pulp Server config
pulp_server_config: "{{
  pulp_server_config__default.update(pulp_server_config__custom) }}{{
  pulp_server_config__default }}"


# Pulp Server Plugins configuration
pulp_server_plugins_config: {}
# Example:
#pulp_server_plugins_config:
#  yum_importer:
#    proxy_host: http://192.168.1.100
#    proxy_port: 3128


# Default option values of the Pulp Workers config
pulp_workers_config_celeryd_log_file: /var/log/pulp/%n.log
pulp_workers_config_celeryd_log_level: INFO
pulp_workers_config_celeryd_pid_file: /var/run/pulp/%n.pid
pulp_workers_config_pythonencoding: UTF-8
pulp_workers_config_pulp_concurrency: "{{ ansible_processor_cores }}"
pulp_workers_config_celery_app: pulp.server.async.app
pulp_workers_config_celery_create_dirs: 1
pulp_workers_config_celeryd_nodes: ""
pulp_workers_config_celeryd_opts: "-c 1 --events --umask=18"
pulp_workers_config_celeryd_user: "{{ pulp_web_service_user }}"
pulp_workers_config_default_nodes: ""

# Default Pulp Workers config options
pulp_workers_config__default:
  celeryd_log_file: "{{ pulp_workers_config_celeryd_log_file }}"
  celeryd_log_level: "{{ pulp_workers_config_celeryd_log_level }}"
  celeryd_pid_file: "{{ pulp_workers_config_celeryd_pid_file }}"
  pythonioencoding: "{{ pulp_workers_config_pythonencoding }}"
  pulp_concurrency: "{{ pulp_workers_config_pulp_concurrency }}"
  celery_app: "{{ pulp_workers_config_celery_app }}"
  celery_create_dirs: "{{ pulp_workers_config_celery_create_dirs }}"
  celeryd_nodes: "{{ pulp_workers_config_celeryd_nodes }}"
  celeryd_opts: "{{ pulp_workers_config_celeryd_opts }}"
  celeryd_user: "{{ pulp_workers_config_celeryd_user }}"
  default_nodes: "{{ pulp_workers_config_default_nodes }}"

# Default Pulp Workers config options
pulp_workers_config__custom: {}

# Final Pulp Workers config
pulp_workers_config: "{{
  pulp_workers_config__default.update(pulp_workers_config__custom) }}{{
  pulp_workers_config__default }}"


# Default values of the options of the server section of the Pulp Admin config
pulp_admin_config_server_host: "{{ pulp_host }}"
pulp_admin_config_server_verify_ssl: "{{ pulp_verify_ssl }}"

# Default options of the server section of the Pulp Admin config
pulp_admin_config_server__default:
  host: "{{ pulp_admin_config_server_host }}"
  verify_ssl: "{{ pulp_admin_config_server_verify_ssl }}"

# Custom options of the server section of the Pulp Admin config
pulp_admin_config_server__custom: {}

# Final server section of the Pulp Admin config
pulp_admin_config_server: "{{
  pulp_admin_config_server__default.update(pulp_admin_config_server__custom) }}{{
  pulp_admin_config_server__default }}"

# Default content of the other section of the Pulp Admin config
pulp_admin_config_client: {}
pulp_admin_config_filesystem: {}
pulp_admin_config_logging: {}
pulp_admin_config_output: {}

# Default Pulp Admin config sections
pulp_admin_config__default:
  server: "{{ pulp_admin_config_server }}"
  client: "{{ pulp_admin_config_client }}"
  filesystem: "{{ pulp_admin_config_filesystem }}"
  logging: "{{ pulp_admin_config_logging }}"
  output: "{{ pulp_admin_config_output }}"

# Custom Pulp Admin config sections
pulp_admin_config__custom: {}

# Final Pulp Admin config
pulp_admin_config: "{{
  pulp_admin_config__default.update(pulp_admin_config__custom) }}{{
  pulp_admin_config__default }}"


# Default values of the options of the server section of the Pulp Consumer config
pulp_consumer_config_server_host: "{{ pulp_host }}"
pulp_consumer_config_server_verify_ssl: "{{ pulp_verify_ssl }}"

# Default options of the server section of the Pulp Consumer config
pulp_consumer_config_server__default:
  host: "{{ pulp_consumer_config_server_host }}"
  verify_ssl: "{{ pulp_consumer_config_server_verify_ssl }}"

# Custom options of the server section of the Pulp Consumer config
pulp_consumer_config_server__custom: {}

# Final server section of the Pulp Customer config
pulp_consumer_config_server: "{{
  pulp_consumer_config_server__default.update(pulp_consumer_config_server__custom) }}{{
  pulp_consumer_config_server__default }}"

# Default content of other sections of the Pulp Consumer config
pulp_consumer_config_authentication: {}
pulp_consumer_config_client: {}
pulp_consumer_config_filesystem: {}
pulp_consumer_config_reboot: {}
pulp_consumer_config_logging: {}
pulp_consumer_config_output: {}
pulp_consumer_config_messaging: {}
pulp_consumer_config_profile: {}

# Default sections of the Pulp Consumer config
pulp_consumer_config__default:
  server: "{{ pulp_consumer_config_server }}"
  authentication: "{{ pulp_consumer_config_authentication }}"
  client: "{{ pulp_consumer_config_client }}"
  filesystem: "{{ pulp_consumer_config_filesystem }}"
  reboot: "{{ pulp_consumer_config_reboot }}"
  logging: "{{ pulp_consumer_config_logging }}"
  output: "{{ pulp_consumer_config_output }}"
  messaging: "{{ pulp_consumer_config_messaging }}"
  profile: "{{ pulp_consumer_config_profile }}"

# Custom sections of the Pulp Customer config
pulp_consumer_config__custom: {}

# Final Pulp Customer config
pulp_consumer_config: "{{
  pulp_consumer_config__default.update(pulp_consumer_config__custom) }}{{
  pulp_consumer_config__default }}"


# Default values of the options of the main section of the Pulp repo_auth config
pulp_repo_auth_config_main_enabled: false
pulp_repo_auth_config_main_log_failed_cert: true
pulp_repo_auth_config_main_log_failed_cert_verbose: false
pulp_repo_auth_config_main_max_num_certs_in_chain: 100
pulp_repo_auth_config_main_verify_ssl: "{{ pulp_verify_ssl }}"

# Default options of the main section of the Pulp repo_auth config
pulp_repo_auth_config_main__default:
  enabled: "{{ pulp_repo_auth_config_main_enabled }}"
  log_failed_cert: "{{ pulp_repo_auth_config_main_log_failed_cert }}"
  log_failed_cert_verbose: "{{ pulp_repo_auth_config_main_log_failed_cert_verbose }}"
  max_num_certs_in_chain: "{{ pulp_repo_auth_config_main_max_num_certs_in_chain }}"
  verify_ssl: "{{ pulp_repo_auth_config_main_verify_ssl }}"

# Custom options of the main section of the Pulp repo_auth config
pulp_repo_auth_config_main__custom: {}

# Final main section of the Pulp repo_auth config
pulp_repo_auth_config_main: "{{
  pulp_repo_auth_config_main__default.update(pulp_repo_auth_config_main__custom) }}{{
  pulp_repo_auth_config_main__default }}"

# Default values of the options of the repos section of the Pulp repo_auth config
pulp_repo_auth_config_repos_cert_location: /etc/pki/pulp/content
pulp_repo_auth_config_repos_global_cert_location: /etc/pki/pulp/content
pulp_repo_auth_config_repos_protected_repo_listing_file: /etc/pki/pulp/content/pulp-protected-repos

# Default options of the repos section of the Pulp repo_auth config
pulp_repo_auth_config_repos__default:
  cert_location: "{{ pulp_repo_auth_config_repos_cert_location }}"
  global_cert_location: "{{ pulp_repo_auth_config_repos_global_cert_location }}"
  protected_repo_listing_file: "{{ pulp_repo_auth_config_repos_protected_repo_listing_file }}"

# Custom options of the repos section of the Pulp repo_auth config
pulp_repo_auth_config_repos__custom: {}

# Final repos section of the Pulp repo_auth config
pulp_repo_auth_config_repos: "{{
  pulp_repo_auth_config_repos__default.update(pulp_repo_auth_config_repos__custom) }}{{
  pulp_repo_auth_config_repos__default }}"

# Default sections of the Pulp repo_auth config
pulp_repo_auth_config__default:
  main: "{{ pulp_repo_auth_config_main }}"
  repos: "{{ pulp_repo_auth_config_repos }}"

# Custom sections of the Pulp repo_auth config
pulp_repo_auth_config__custom: {}

# Final Pulp repo_auth config
pulp_repo_auth_config: "{{
  pulp_repo_auth_config__default.update(pulp_repo_auth_config__custom) }}{{
  pulp_repo_auth_config__default }}"
```


Dependencies
------------

- [`config_encoder_filters`](https://github.com/jtyr/ansible-config_encoder_filters)
- [`mongodb`](https://github.com/jtyr/ansible-mongodb) role (optional)
- [`qpid_cpp_server`](https://github.com/jtyr/ansible-qpid_cpp_server) role (optional)
- [`rabbitmq`](https://github.com/jtyr/ansible-rabbitmq) role (optional)


License
-------

MIT


Author
------

Jiri Tyr
