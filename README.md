Ansible Atlassian BambooAgent role
==================================

This role performs the necessary for running an Atlassian Bamboo remote agent on a target machine.


The role performs the following actions:

* creates the user running the bamboo agent,
* installs the certificate of the bamboo server such that it is possible
  to download the bamboo agent `jar` file directly from the Server without bypassing any security (optional).
* creates the startup scripts for the bamboo agent, populates additional paths (build tools) and additional options
  (either from the system like `CUDA_VISIBLE_DEVICES` or additional Bamboo Agent configurations)
* registers the startup script of the bamboo agent for launching the agent at boot time.
* registers an auto start service on the operating system
* populates the capabilities of the agent
* changes the build folder and the name of the agent


Requirements
------------
Java should be installed on the target operating system. Consider using the `ansible-atlassian-bambooagent-oracle-java` role for this.


Role Variables
--------------

The following variable need to be set for the role.

| variable | default | meaning |
|----------|---------|---------|
| `bamboo_server_url` | `""` (empty string) | Indicates the URL of your Bamboo instance. Should be set.|
| `bambooagent_user` | `bambooagent` | the user running the bamboo agent|
| `bambooagent_group`| `bambooagent_grp` | the group which the bamboo agent user is in|
| `bambooagent_service_name` | `bambooagent` | the name of the service running the bamboo agent. This will appear as the service for starup-shutdown admin commands|
| `bambooagent_install_root`| `/home/{{ bambooagent_user }}` | the root folder under which all the programs/scripts of the agent (starters, other local programs) will be installed. This can be the home folder of the agent, although it will contain the build folder under `bambooagent_agent_root`.|
| `bambooagent_agent_root`| `{{ bambooagent_install_root }}/bamboo-agent-home` | the root folder for the files specific for running the bamboo agent (the .jar file, wrapper, etc).|
| `bambooagent_version` | 5.11.1.1 | the version of the agent |
| `bamboo_java_jar_file` | "" (empty string) | The `.jar` of the Bamboo agent launcher. If empty (default), the role will attempt to fetch this file from the Bamboo server directly. Note that this refers to the one with the service wrapper. |
| `bambooagent_jar_filename` | `atlassian-bamboo-agent-installer-{{ bambooagent_version }}.jar` | the jar file **on the remote agent** |
| `bambooagent_jar_filename_full_path`| `{{ bambooagent_install_root }}/{{ bambooagent_jar_filename }}` | the full path location of the jar file **on the remote agent** |
| `bambooagent_capability_file`| `{{ bambooagent_agent_root }}/bin/bamboo-capabilities.properties` | the location of the capabilities file on the remote |
| `bambooagentjava_additional_options`| <ul><li>`-Djava.awt.headless=true`</li><li>`-Dbamboo.home={{ bambooagent_agent_root }}`</li></ul> |additional options passed to the Java virtual machine. This should be a list|
| `bambooagent_additional_environment`| `[]` (empty list) | additional environment variables set before running the bamboo agent (eg. `CUDA_VISIBLE_DEVICES=1`). This should be a list |
|`certificate_files`| `[]` | Certificates definition list (see below).|
|`bamboo_verify_certificates`| `True` | verifies the server certificates when fetching the JAR file from it. |

### Java
The version of the agent should work well with the installed Java. For instance,version 5.11 of the Bamboo agent require Java 8. The `JAVA_HOME` is set automatically on OSX during agent's startup.

### Bamboo capability
Specific capability may be declared automatically by the agent using a feature of Atlassian Bamboo: the capability file.
This file has a very simple format and lies inside the installation folder.

The *capability* file will receive the capabilities that are declared by running the playbook. This is a list of pairs (dictionary) indicating the name
of the capability along with its value.

```
- name: '[BAMBOO] empty capabilities declaration'
  set_fact:
    bamboo_capabilities: {}
```

It is possible to update the capabilities first by reading those from disk, using eg a `pre_task`:

```
pre_tasks:
    - name: Reading the agent capability file
      include_role:
        name: atlassian_bambooagent_role
        tasks_from: read_capability
```

and then write them on disk as eg. a `post_task`:

```
post_tasks:
  - name: Updating the agent capability file
    include_role:
      name: atlassian_bambooagent_role
      tasks_from: write_capability
```

The read and write are both using the dictionary `bamboo_capabilities` (as a `fact`) as input/output. The functions take care
of the escaping of the `/` and `\` properly on the different platforms.

A typical play in a playbook would look like this:

```
- hosts: my-bamboo-agents
  vars:
    # this variable needs to be set to retrieve the capability file
    - bambooagent_agent_root: "the specified agent root or the default"

  pre_tasks:
    # this will read the capability file if it exist
    - name: Reading the agent capability file
      include_role:
        name: atlassian_bambooagent_role
        tasks_from: read_capability

  post_tasks:
    # this will update the capability file and create it if needed
    - name: Updating the agent capability file
      include_role:
        name: atlassian_bambooagent_role
        tasks_from: write_capability

  tasks:
    # ... tasks

    - name: 'update capability'
      set_fact:
        bamboo_capabilities: "{{ bamboo_capabilities | combine({item.key:item.value}) }}"
      loop:
        - key: 'bamboo_custom_capability'
          value: "bamboo_custom_capability_value"
        # ...

```

### Removing capabilities
Over time, being able to maintain the capabilities is important, especially when the number of agents increases.
Using the same tools as above, it is possible to remove capabilities that became obsolete. This can be done by
indicating in the list `bamboo_capabilities_to_remove` the names of the capabilities that need to be removed.

```
- hosts: my-bamboo-agents
  vars:
    - bambooagent_agent_root: "the specified agent root or the default"

  tasks:
    - name: '[BAMBOO] remove obsolete capabilities'
      set_fact:
        bamboo_capabilities_to_remove:
          - cache_folder1
          - qt_version3

  post_tasks:
    # this will update the capability file and create it if needed
    - name: Updating the agent capability file
      include_role:
        name: atlassian_bambooagent_role
        tasks_from: write_capability

```

### Getting the agent UUID

The role contains a dedicated helper to retrieve the agent's `UUID`, which makes it easier
to manage the approved agents in the Bamboo agents' admin view. This can be used like the following
example, which will fill the variable `bamboo_agent_UUID`.

```
- hosts: my-bamboo-agents
  vars:
    - bambooagent_agent_root: "the specified agent root or the default"

  tasks:
    - name: Retrieves Agent UUID
      include_role:
        name: atlassian_bambooagent_role
        tasks_from: get_agent_uuid
      tags: ['never', 'bamboo_agent_uuid']

    - name: Print agent UUID
      debug:
        var: bamboo_agent_UUID
      tags: ['never', 'bamboo_agent_uuid']
```

### HTTPS certificate to the service
The certificate should be in the variable `certificate_files` (a list of certificates which are alias/filename pairs) like the following:

```
- certificate_files:
  - alias: "bamboo_server_certificate_alias"
    file: "{{some_root_location}}/bamboo_server_certificate.crt"
```

### Build folder
The build folder can be changed after installation and proper registration of the agent with the Bamboo server.
This is particularly relevant in the following scenarios:

* the agent is installed on Windows: the lenght of the paths matters for most of the tools you would be using,
  shortening the prefix of the path is important. It is then possible to install the agent in some location, and then
  point the build folder to a folder directly under the root of a partition
* when you want to separate the build data from the agent's data and configuration: you may then use different disks
  drives (fast ones for the build folder, smaller ones for the agent's), or have a separate backup policy on those
  folders.

As mentioned, the build folder can be set only after the agent has been properly installed and registered with the Bamboo server. After this, the correct folder structure and configuration file appears in the installation folder and it is
possible to change the build folder.

An example of changing the build folder would be this:
```
- hosts: windows-agents
  vars:
    - bambooagent_agent_root: "{{ bamboo_agents[inventory_hostname].bamboo_folder }}"

  tasks:
    - name: Updating the agents' configuration
      include_role:
        name: atlassian_bambooagent_role
        tasks_from: update_agent_configuration
      vars:
        bamboo_agent_name: "new-name-for-the-agent"
        bamboo_agent_description: "Remote agent on XYZ"
        bamboo_build_folder: "D:\\"
        bambooagent_user: "bamboo_user" # optional to create the build folder with proper rights
      tags: ['never', 'update_bamboo_config']
```

The previous task will not be run except specified explicitely on the command line. It is better to stop the service
prior to running this update. From the command line, it can be achieved like this, or fully integrated in a play:

```
ansible \
  remote-machine-name-or-group \
  -m win_service \
  -a "name=bamboo-remote-agent state=stopped" \
  --inventory inventory-bamboo.yml \
  --become

# we modify some of the installation settings
ansible-playbook \
  --limit remote-machine-name-or-group\
  --inventory inventory-bamboo.yml \
  --become \
  --tags=update_bamboo_config \
  playbooks/my-windows-play.yml

# we restart the service again
ansible \
  remote-machine-name-or-group \
  -m win_service \
  -a "name=bamboo-remote-agent state=restarted" \
  --inventory inventory-bamboo.yml \
  --become
```

Note that some of the updated fields will not appear on the server. Removing the agent from the server and
re-registering it afterwards should do (known bug in Bamboo agents).

Dependencies
------------

No additional dependency.

Example Playbook
----------------


```
- hosts: bambooagents

  vars:
    - program_location: /folder/containing/installers/
    - server_url: https://my.local.network/bamboo/

    # the home folder of the Bamboo agent (for examples, should be different on Linux/OSX/etc)
    - local_var_bambooagent_install_root: "/somebigdrive/bambooagent"

    # used to compute the name of the bamboo agent JAR file to transfer to the remote
    - bambooagent_version: "6.8.1"

    # this points to a local copy of the .jar that can be downloaded from the server.
    - local_copy_of_bamboo_agent_jar: "/some/folder/{{ bambooagent_jar_filename }}"

  pre_tasks:
    # Reads capabilities if it already exists, otherwise returns an empty dict
    - name: Reading the agent capability file
      include_role:
        name: atlassian_bambooagent_role
        tasks_from: read_capability

  post_tasks:
    # Writes the capabilities back to file
    - name: Updates agent capabilities
      include_role:
        name: atlassian_bambooagent_role
        tasks_from: write_capability

  roles:
    # This installs the bamboo agent, and overrides the variables
    - name: installing the bamboo agent
      role: atlassian_bambooagent_role
      vars:
        bambooagent_user: "bamboo_service_user"
        bambooagent_group: "bamboo_service_group"
        bambooagent_agent_root: "/mount/folder/fast/disk/bamboo-agent"
        bambooagent_service_name: atlassian-bambooagent
        bamboo_server_url: "{{ server_url }}"
        bamboo_java_jar_file: "{{ local_copy_of_bamboo_agent_jar }}"
        bambooagent_install_root: "{{ local_var_bambooagent_install_root }}"
        certificate_files:
          - alias: "my.certificate.authority.crt"
            file: "/some/local/folder/my.certificate.authority.crt"
      tags: bamboo


  tasks:
    # Example for declaring custom capabilities
    - name: '[BAMBOO] default capabilities'
      set_fact:
        bamboo_capabilities: "{{ bamboo_capabilities | combine({item.key:item.value}) }}"
      loop:
        - key: 'operating_system'
          value: "{{ bamboo_operating_system }}"
        - key: agent_name
          value: "{{ ansible_fqdn }}"
        - key: osversion
          value: "{{ ansible_distribution_version.split('.')[:2] | join('.') }}"

    # Example declaring system builder capabilities (python binary, already installed)
    - block:
      - name: '[BAMBOO] running python'
        command: python -c "import sys; print('%d.%d\n%s' % (sys.version_info.major, sys.version_info.minor, sys.executable))"
        register: bamboo_capability_python_version

      - name: '[BAMBOO] register python'
        set_fact:
          bamboo_capabilities: "{{ bamboo_capabilities | combine({item.key:item.value}) }}"
        loop:
          - key: 'system.builder.command.python{{bamboo_capability_python_version.stdout_lines.0}}'
            value: '{{ bamboo_capability_python_version.stdout_lines.1 }}'
```

License
-------

BSD

Author Information
------------------

Any comments on the Ansible, PR or bug reports are welcome from the corresponding Github project.

Change log
----------

## 0.1
* first official version (not really, but previous releases did not have changelogs)
* Change of role name to `atlassian_bambooagent_role` to follow [those guidelines](https://docs.ansible.com/ansible/devel/dev_guide/developing_collections.html#roles-directory)
* additional linting
* new option `bamboo_verify_certificates` to avoid checking the server certificate when fetching the JAR from Bamboo. This is
  useful on OSX only (see [here](https://stackoverflow.com/a/56031239/1617295)) when the server has a public certificate.
  In case the server uses its own CA, that CA is already installed system wide by the role.
* bug fix on Windows when fetching the JAR from the Bamboo server