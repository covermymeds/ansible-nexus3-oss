# Ansible Role: Nexus 3 OSS

This role installs and configures Nexus Repository Manager OSS version 3.x on CentOS/RHEL.

All configuration can be updated by re-running the role, except for the [blobstores](https://books.sonatype.com/nexus-book/3.0/reference/admin.html#admin-repository-blobstores)-related settings, which are immutable in nexus.

## Requirements

- This has only been tested on CentOS 7
- Oracle Java 8 (mandatory)
- Apache HTTPD (optional, used to setup a SSL reverse-proxy)

(see [Dependencies](#dependencies) section below for matching roles on galaxy)

## Role Variables


Ansible variables, along with the default values (see `default/main.yml`) :

    nexus_version: '3.1.0-04'
    nexus_timezone: 'UTC'
    nexus_package: "nexus-{{ nexus_version }}-unix.tar.gz"

The nexus version and package to install, see available versions at https://www.sonatype.com/download-oss-sonatype . `nexus_timezone` is a Java Timezone name and can be useful in combination with `nexus_scheduled_tasks` cron expressions below.

    nexus_download_dir: '/tmp'

Directory on target where the nexus package will be downloaded.

    nexus_os_group: 'nexus'
    nexus_os_user: 'nexus'

User and group used to own the nexus files and run the service, those will be created by the role if absent.

    nexus_installation_dir: '/opt'
    nexus_data_dir: '/var/nexus'
    nexus_tmp_dir: '/tmp/nexus'

Nexus directories, `nexus_installation_dir` contains the installed executable(s), `nexus_data_dir` contains all configuration, repositories and uploaded artifacts. Note: custom blobstores paths outside of `nexus_data_dir` can be configured, see `nexus_blobstores` below.

    nexus_admin_password: 'changeme'

The 'admin' account password to setup. Note : admin password change subsequent to first-time provisionning/install is *not implemented* by this role yet.

    nexus_anonymous_access: false

Allow [anonymous access](https://books.sonatype.com/nexus-book/3.0/reference/security.html#anonymous) to nexus.

    public_hostname: 'nexus.vm'

The fully qualified domain name under which the nexus instance will be accessible to its clients.

    nexus_branding_header: ""
    nexus_branding_footer: "Last provisionned {{ ansible_date_time.iso8601 }}"

Header and footer branding, those can contain HTML.

    nexus_bind_address: 127.0.0.1

If configuring a reverse proxy, bind nexus to localhost only.  By default nexus
will listen on all interfaces (0.0.0.0).

    ldap_connections: []

[LDAP connection(s)](https://books.sonatype.com/nexus-book/3.0/reference/security.html#ldap) setup, each item goes as follow :

```
  - ldap_name: 'My Company LDAP' # used as a key to update the ldap config
    ldap_protocol: 'ldaps' # ldap or ldaps
    ldap_hostname: 'ldap.mycompany.com'
    ldap_port: 636
    ldap_search_base: 'dc=mycompany,dc=net'
    ldap_user_base_dn: 'ou=users'
    ldap_user_object_class: 'inetOrgPerson'
    ldap_user_id_attribute: 'uid'
    ldap_user_real_name_attribute: 'cn'
    ldap_user_email_attribute: 'mail'
    ldap_group_base_dn: 'ou=groups'
    ldap_group_object_class: 'posixGroup'
    ldap_group_id_attribute: 'cn'
    ldap_group_member_attribute: 'memberUid'
    ldap_group_member_format: '${username}'
```


    nexus_privileges:
      - name: all-repos-read # used as key to update a privilege
        description: 'Read & Browse access to all repos'
        repository: '*'
        actions: # can be add, browse, create, delete, edit, read or  * (all)
          - read
          - browse

List of the [privileges](https://books.sonatype.com/nexus-book/3.0/reference/security.html#privilegeshttps://books.sonatype.com/nexus-book/3.0/reference/security.html#privileges) to setup. Those items are combined with the following default values :

```
    _nexus_privilege_defaults:
      type: repository-view
      format: maven2
      actions:
        - read
```

```
    nexus_roles:
      - id: Developpers # can map to a LDAP group id, also used as a key to update a role
        name: developers
        description: All developers
        privileges:
          - nx-search-read
          - all-repos-read
        roles: [] # references to other role names
```

List of the [roles](https://books.sonatype.com/nexus-book/3.0/reference/security.html#roles) to setup.

    nexus_local_users: []

Local (non-LDAP) users/accounts to create in nexus, items go as follow :

```
  - username: jenkins # used as key to update
    first_name: Jenkins
    last_name: CI
    email: support@company.com
    password: "s3cr3t"
    roles:
      - developers # role ID
```

    nexus_delete_default_repos: false

Delete the repositories from the nexus install initial default configuration. This step is only executed on first-time install (when `nexus_data_dir` has been detected empty).

    nexus_delete_default_blobstore: false

Delete the default blobstore from the nexus install initial default configuration. This can be done only if `nexus_delete_default_repos: true` and all configured repositories (see below) have an explicit `blob_store: custom`. This step is only executed on first-time install (when `nexus_data_dir` has been detected empty).

    nexus_blobstores: []
    # example blobstore item :
    # - name: separate-storage
    #   path: /mnt/custom/path
    
[Blobstores](https://books.sonatype.com/nexus-book/3.0/reference/admin.html#admin-repository-blobstores) to create. A blobstore path and a repository blobstore cannot be updated after initial creation (any update here will be ignored on re-provisionning).

    nexus_scheduled_tasks: []
    #  example task to compact blobstore :
    #  - name: compact-blobstore
    #    cron: '0 0 22 * * ?'
    #    typeId: blobstore.compact
    #    taskProperties:
    #      blobstoreName: 'default' # all task attributes are stored as strings by nexus internally

[Scheduled tasks](https://books.sonatype.com/nexus-book/reference3/admin.html#admin-system-tasks) to setup. `typeId` and task-specific `taskProperties` can be guessed either from the java type hierarchy of `org.sonatype.nexus.scheduling.TaskDescriptorSupport` or from peeking at the browser AJAX requests while manually configuring a task.

    nexus_repos_maven_proxy:
      - name: central
        remote_url: 'https://repo1.maven.org/maven2/'
        layout_policy: permissive
      - name: jboss
        remote_url: 'https://repository.jboss.org/nexus/content/groups/public-jboss/'
    # example with a login/password :
    # - name: secret-remote-repo
    #   remote_url: 'https://company.com/repo/secure/private/go/away'
    #   remote_username: 'username'
    #   remote_password: 'secret'

Maven [proxy repositories](https://books.sonatype.com/nexus-book/3.0/reference/maven.html#_proxying_maven_repositories) configuration.


    nexus_repos_maven_hosted:
      - name: private-release
        version_policy: release
        write_policy: allow_once

Maven [hosted repositories](https://books.sonatype.com/nexus-book/3.0/reference/maven.html#_hosting_maven_repositories) configuration.

    nexus_repos_maven_group:
      - name: public
        member_repos:
          - central
          - jboss

Maven [group repositories](https://books.sonatype.com/nexus-book/3.0/reference/maven.html#_grouping_maven_repositories) configuration.

All three repository types are combined with the following default values :

```
    _nexus_repos_maven_defaults:
      blob_store: default # Note : cannot be updated once the repo has been created
      strict_content_validation: true
      version_policy: release # release, snapshot or mixed
      layout_policy: strict # strict or permissive
      write_policy: allow_once # allow_once or allow
```


## Dependencies

This role requires Ansible 2.1 or higher.

The java and httpd requirements /can/ be fulfilled with the following galaxy roles :
  - [ansiblebit.oracle-java](https://galaxy.ansible.com/ansiblebit/oracle-java/)
  - [geerlingguy.apache](https://galaxy.ansible.com/geerlingguy/apache/)

## Example Playbook

```
---
- name: Nexus
  hosts: nexus
  become: yes

  vars:
    nexus_version: '3.1.0-04'
    nexus_timezone: 'Canada/Eastern'
    nexus_admin_password: "{{ vault_nexus_admin_password }}"
    ldap_connections:
      - ldap_name: 'Company LDAP'
        ldap_protocol: 'ldaps'
        ldap_hostname: 'ldap.company.com'
        ldap_port: 636
        ldap_search_base: 'dc=company,dc=net'
        ldap_user_base_dn: 'ou=users'
        ldap_user_object_class: 'inetOrgPerson'
        ldap_user_id_attribute: 'uid'
        ldap_user_real_name_attribute: 'cn'
        ldap_user_email_attribute: 'mail'
        ldap_group_base_dn: 'ou=groups'
        ldap_group_object_class: 'posixGroup'
        ldap_group_id_attribute: 'cn'
        ldap_group_member_attribute: 'memberUid'
        ldap_group_member_format: '${username}'
    nexus_privileges:
      - name: all-repos-read
        description: 'Read & Browse access to all repos'
        repository: '*'
        actions:
          - read
          - browse
      - name: company-project-deploy
        description: 'Deployments to company-project'
        repository: company-project
        actions:
          - add
          - edit
    nexus_roles:
      - id: Developpers # maps to the LDAP group
        name: developers
        description: All developers
        privileges:
          - nx-search-read
          - all-repos-read
          - company-project-deploy
        roles: []
    nexus_local_users:
      - username: jenkins # used as key to update
        first_name: Jenkins
        last_name: CI
        email: support@company.com
        password: "s3cr3t"
        roles:
          - Developpers # role ID here
    nexus_blobstores:
      - name: company-artifacts
        path: /var/nexus/blobs/company-artifacts
    nexus_scheduled_tasks:
      - name: compact-blobstore
        cron: '0 0 22 * * ?'
        typeId: blobstore.compact
        taskProperties:
          blobstoreName: 'company-artifacts'
    nexus_repos_maven_proxy:
      - name: central
        remote_url: 'https://repo1.maven.org/maven2/'
        layout_policy: permissive
      - name: alfresco
        remote_url: 'https://artifacts.alfresco.com/nexus/content/groups/private/'
        remote_username: 'secret-username'
        remote_password: "{{ vault_alfresco_private_password }}"
      - name: jboss
        remote_url: 'https://repository.jboss.org/nexus/content/groups/public-jboss/'
      - name: vaadin-addons
        remote_url: 'https://maven.vaadin.com/vaadin-addons/'
      - name: jaspersoft
        remote_url: 'https://jaspersoft.artifactoryonline.com/jaspersoft/jaspersoft-repo/'
        version_policy: mixed
    nexus_repos_maven_hosted:
      - name: company-project
        version_policy: mixed
        write_policy: allow
        blob_store: company-artifacts
    nexus_repos_maven_group:
      - name: public
        member_repos:
          - central
          - jboss
          - vaadin-addons
          - jaspersoft

  roles:
    - role: ansiblebit.oracle-java
      oracle_java_set_as_default: yes
    - role: geerlingguy.apache
      apache_create_vhosts: no
    - role: savoirfairelinux.nexus3-oss

```

License
-------

GNU GPLv3

Author Information
------------------

See https://github.com/savoirfairelinux/ansible-nexus3-oss
