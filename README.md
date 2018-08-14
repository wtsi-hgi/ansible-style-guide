# Ansible Style Guide
When creating a large Ansible configuration with a team, it is important to align to a set of conventions in order to
achieve a setup that is easy to maintain and extend. This guide is intended to complement
[the best practice guidelines supplied by Ansible](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html),
with additional rules to help when a configuration becomes big, complex and is managed by multiple developers.


## Components
### Roles
#### Name
Role names must follow kebab-case and should be in singular form.

#### Location
Roles must be placed inside the `roles` directory. All variables that can be used to configure the role must be
mentioned in the `defaults/main.yml` file. If it is not appropriate for a default value to be set, the variable is to be
listed at the top of the file in a comments section and should be in the form:
```yaml
# Variables that have no defaults and must be set:
#   $variable_name:
#     [($python_style_data_type)] $description
```
For example:
```
# Variables that have no defaults and must be set:
#   hail_jupyter_token:
#     (str) an authentication token used to login to Jupyter notebook
```
A task should be added to the start of the role to check that all required variables have been set. The `assert` module
could be used for this purpose (remember to use `no_log` or `msg` appropriately to prevent secrets from being
accidentally spilled).

#### Implementation
##### Interfaces
There must be a clean interface through which roles are configured; role variables are to be set in plays, such that
all values used to configure the role are binded to role parameters via the associated `vars` section, e.g. in the
`hailers.yml` playbook:
```yaml
- hosts: hailers
  roles:
    - role: hail
      vars:
        hail_version: "{{ hail_GROUP_version }}"
        hail_cluster_name: "{{ hail_SPECIFIC_GROUP_name }}"
```
A role must not "reach out" and use variables that happen to be in scope, as this can both couple roles to specific
setups, and it can to make it difficult to know where a value used in a role has been defined. Roles must therefore only
use variables mentioned in `default` or `vars` files (with the exception of the use of built-ins).

##### Handlers
By default, handlers are executed at the end of a play. This is problematic if a failure occurs, as the triggering
change may not occur on the re-run, and subsequently the handler may never fire. For this reason, use of Ansible
handlers should be kept to a minimum.

Note: the same problem can occur with registering variables and executing subsequent actions based on whether a change
occurred.

##### Re-use
It is good practice to design roles such that they could be reused by others. Ideally, such roles would be developed and
tested in separate repositories, then imported using [Ansible Galaxy](https://galaxy.ansible.com/) or by use of
[git-subrepo](https://github.com/ingydotnet/git-subrepo) (or similar). However, this will not be appropriate for some,
very project specific roles.

### Playbooks
#### Name
Playbook names must follow kabab-case and should be in plural form.

#### Implementation
It is often useful for playbooks to first assert hosts are members of the correct groups - this will lead to fewer
variable undefined related surprises later.

### Groups
#### Name
There are three distinct types of Ansible group:
1) Groups containing variables that are used in a single playbook (e.g. configurations in the `irobots` group are used
exclusively with the `irobots.yml` playbook). These groups should take the same name as the playbook, which should be
in plural form.
2) Groups of general configuration values, which are used in many playbooks (e.g. configurations for your GitLab
account, such as `gitlab_url`, `gitlab_token`). Such groups should have a relevant name and be in singular form.
3) Groups containing variables that are specific for a set of hosts, which shall be referred to as "specific groups".
For example, the group `hail-cluster-bob` could define variables used by Bob's Hail cluster. These specific groups
should have a relevant prefix, followed by the name of the cluster. The name should be in singular form.

All group names must be kebab-case.

#### Location
All files containing group variables must to be located in sub-directories in `group_vars`. Plaintext variables must
be defined in a file called `vars.yml` and secret variables are to be placed in an
[Ansible Vault](http://docs.ansible.com/ansible/latest/user_guide/vault.html) named `vault`. To aid searchability, all
variables defined in Ansible vaults must be referenced in the header of the associated `vars.yml` file, in the format:
```
# Value contains:
#   example_GROUP_variable
```


### Tasks
Tasks should always have a descriptive name. The first word should not be capitalised by default but capital letters may
be used for proper nouns and acronyms. The name does not need to be prefixed with the role name and should not be
constructed programmatically.


## Variables
Ansible setups can defines hundreds of variables, which can be changed to customise the setup. When using Ansible at
scale, it is important to follow naming conventions to be able to easily work with so many variables.

All Ansible variables must be namespaced to enable developers to easily find where they have been defined and to avoid
naming collisions. By looking at the name alone, it should be possible to know what file(s) the variable has been
defined in, without false positives.

Variables must be:
- Descriptive and not use obscure acronyms or abbreviations.
- Snake case (i.e. `my_example_variable`).
- Prefixed with the role/group/play to which the variable belongs. For example, variables for the role `hail` are to be
in the form `hail_*`, such as `hail_version`, `hail_ssl_key_file`).
- Started with `_` if they are meant to be private to the defining file (private variables must still follow all other
rules).
- Indicative as to where they have been defined, according to the rules in the table below:
    <table>
        <tbody>
            <tr>
                <th>Type</th>
                <th>Name</th>
                <th>Description</th>
                <th>Defined In</th>
            </tr>
            <tr>
                <td>Role</td>
                <td><code>*_*</code></td>
                <td>
                    The prefix matches the name of the role, followed by an underscore, then a suffix that describes the
                    variable's purpose.
                </td>
                <td><code>roles/$0/defaults/main.yml</code>, <code>roles/$0/vars/main.yml</code></td>
            </tr>
            <tr>
                <td>Group</td>
                <td><code>*_GROUP_*</code></td>
                <td>
                    <p>
                        The prefix matches the snake-case version of the group name, followed by <code>_GROUP_</code>,
                        then a suffix that describes the variable's purpose.
                    </p>
                    <p>
                        Often, the suffix will match that of the role variable to which the group variable
                        corresponds. For example, the value of the group variable <code>hailers_GROUP_version</code> may
                        be binded to the <code>hail_version</code> role parameter.
                    </p>
                </td>
                <td><code>group_vars/$0/vars.yml</code>, <code>group_vars/$0/vault</code></td>
            </tr>
            <tr>
                <td>Specific Group</td>
                <td><code>*_SPECIFIC_GROUP_*</code></td>
                <td>
                    <p>
                        The prefix matches the snake-case version of the name of the specific group, followed by
                        <code>_SPECIFIC_GROUP_</code>, then a suffix that describes the variable's purpose.
                    </p>
                    <p>
                        For example, the specific group <code>consul-cluster-hgi</code> may define the variable:
                        <code>consul_cluster_SPECIFIC_GROUP_id: hgi</code>.
                    </p>
                </td>
                <td><code>group_vars/$0s/vars.yml</code>, <code>group_vars/$0s/vault</code></td>
            </tr>
            <tr>
                <td>Playbook</td>
                <td><code>*_PLAYBOOK_*</code></td>
                <td>Defined in the <code>vars</code> section of a play, with a name prefix matching the snake-case
                version of the playbook name (in singular form), followed by <code>_PLAYBOOK_</code>, then a suffix that
                describes the variable's purpose. In general, playbook variables should only be defined if they are to
                be used multiple times.</td>
                <td><code>$0s.yml</code></td>
            </tr>
            <tr>
                <td>Host</td>
                <td><code>*_HOST_*</code></td>
                <td>The prefix should relate to the general purpose of the variable, followed by <code>_HOST_</code>,
                then a suffix that describes the variable's purpose.</td>
                <td><code>host_vars/$0s.yml</code></td>
            </tr>
            <tr>
                <td>Playbook Fact</td>
                <td><code>*_PLAYBOOK_FACT_*</code></td>
                <td>
                    <p>
                        A playbook fact is a variable defined in a playbook with <code>set_fact</code>, intended for use
                        outside of the play where it was set. Use of such variables should generally be kept to a
                        minimum.
                    </p>
                    <p>
                        The prefix should match the snake-case version of the name of the play, followed by
                        <code>_PLAYBOOK_FACT_</code>, then a suffix that describes the variable's purpose.
                    </p>
                </td>
                <td><code>$0s.yml</code></td>
            </tr>
            <tr>
                <td>Role Fact</td>
                <td><code>*_ROLE_FACT_*</code></td>
                <td>
                    <p>
                        A role fact is a variable defined in a role with <code>set_fact</code>, intended for use outside
                        of that role. Use of such variables should generally be kept to a minimum.
                    </p>
                    <p>
                        The prefix should match the name of the role, followed by <code>_role_FACT_</code>, then a
                        suffix that describes the variable's purpose.
                    </p>
                </td>
                <td><code>roles/$0/tasks/*.yml</code></td>
            </tr>
        </tbody>
    </table>
