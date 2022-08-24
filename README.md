ansible_role_nrpe
==============

This role performs the following actions:

* Installs EPEL repositories through meta/dependencies.
* Installs nrpe
* Install nagions-plugins
* Configures nrpe
* Installs optional custom scripts for checks to /usr/local/scripts.

Requirements
---------------

This role has been tested on the following operatingsystems:

* CentOS 7
* RedHat 7

Role variables
---------------

This section describes variables used in the role:

* `ansible_role_nrpe_extra_packages`

List of extra packages to install, should contain a list of nagios plugins needed for checks.

* `ansible_role_nrpe_port`

Listening port for nrpe daemon, must be an non-privileged port (i.e > 1024).

* `ansible_role_nrpe_hosts`

List of hosts allowed to contact the nrpe daemon, IP or or IP/bit mask (i.e. 192.168.1.0/24) are supported

* `ansible_role_nrpe_command_timeout`

Timeout in seconds for nrpe commands, after timeout has expired nrpe will kill the command process.

* `ansible_role_nrpe_connection_timeout`

Timeout in seconds for establishing connections to nrpe.

* `ansible_role_nrpe_queuesize`

Listen queue size (backlog) for serving incoming connections, increase this value if load is high.

* `ansible_role_nrpe_server_address`

Address that nrpe should bind to in case there are more than one interface, use "{{ ansible_default_ipv4.address }}" to use primary IP.

* `ansible_role_nrpe_commands`

A list of commands, paths and arguments to be added to /etc/nrpe.d/custom_commands.cfg. The role will replace this file with contents from these variables. See sample configuration for syntax.

* `ansible_role_nrpe_scripts:`

A list of custom scripts to be added to /usr/local/scripts. name sets the name of the scripts, state should be set to absent or present and data contains the script. Data variable should be followed by | then the script on the following line. See sample for syntax

Sample configuration
---------------

This section contains a sample configuration for this role.

```yaml
ansible_role_nrpe_extra_packages:
  - nagios-plugins-procs
  - nagios-plugins-users
  - nagios-plugins-swap
  - nagios-plugins-load
  - nagios-plugins-disk
ansible_role_nrpe_port: 5666
ansible_role_nrpe_hosts: 127.0.0.1,::1
ansible_role_nrpe_command_timeout: 60
ansible_role_nrpe_connection_timeout: 300
ansible_role_nrpe_queuesize: 5
ansible_role_nrpe_server_address: "{{ ansible_default_ipv4.address }}"
ansible_role_nrpe_commands:
  - command: 'users'
    command_path: '/usr/lib64/nagios/plugins/check_users'
    command_arguments: '-w 5 -c 10'
  - command: 'zombie_procs'
    command_path: '/usr/lib64/nagios/plugins/check_procs'
    command_arguments: '-w 5 -c 10 -s Z,X'
  - command: 'total_procs'
    command_path: '/usr/lib64/nagios/plugins/check_procs'
    command_arguments: '-w 150 -c 200'
  - command: 'load'
    command_path: '/usr/lib64/nagios/plugins/check_load'
    command_arguments: '-w 15,10,5 -c 30,25,20'
  - command: 'root_disk'
    command_path: '/usr/lib64/nagios/plugins/check_disk'
    command_arguments: '-w 20% -c 10% -p /dev/mapper/centos-root'
  - command: 'swap'
    command_path: '/usr/lib64/nagios/plugins/check_swap'
    command_arguments: '-w 40% -c 20%'
  - command: 'proc_crond'
    command_path: '/usr/lib64/nagios/plugins/check_procs'
    command_arguments: '-w 1: -c 1:5 -C crond'
  - command: 'graylog_check_lifecycle'
    command_path: /usr/local/scripts/graylog-check.sh
    command_arguments: lifecycle
  - command: 'graylog_check_lb_status'
    command_path: /usr/local/scripts/graylog-check.sh
    command_arguments: lb_status
  - command: 'graylog_check_is_processing'
    command_path: /usr/local/scripts/graylog-check.sh
    command_arguments: is_processing
ansible_role_nrpe_scripts:
  - name: graylog-check.sh
    state: present
    data: |
      #!/bin/bash
      #####################################################################################################
      #
      # ABOUT THIS PROGRAM
      #
      # NAME
      #   graylog-check.sh -- NRPE script for Graylog health check.
      #
      # AUTHOR
      #   Mattias Jonsson
      #
      # SYNOPSIS
      #   /bin/bash ./graylog-check.sh [parameter]
      #
      #   Parameter must be one of the following: lifecycle, lb_status, is_processing.
      #   Example: /bin/bash ./graylog-check.sh DB
      #
      #   The script returns one of the following statuses:
      #   0 - Service state is ok.
      #   2 - Service state is critical.
      #   3 - Service state is unknown.
      #
      ####################################################################################################

      set -u

      declare api_user=""
      declare api_password=""
      # declare api_token=""
      declare health_status_json=""
      declare health_status=""
      declare int_addr=`/sbin/ifconfig ens160 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}'`

      health_check_func () {
        health_status_json=`/usr/bin/curl -s -u "${api_user}":"${api_password}" -H 'Accept: application/json' -XGET "http://"${int_addr}":9000/api/system?pretty=true"`
        # Check if health_status_json is proper JSON before trying to parse
        if `/bin/echo "${health_status_json}" | /usr/bin/python -m json.tool >/dev/null 2>&1`; then
          health_status=`/bin/echo "${health_status_json}" | grep "$1" | sed 's/[^a-z:]//g' | cut -d ":" -f2`
          case "$health_status" in
            running|alive|true)
              /bin/echo "OK - $1 is ${health_status}."
              exit 0
              ;;
            '')
              /bin/echo "UNKOWN - $1 status is UNKNOWN."
              exit 3
              ;;
            *)
              /bin/echo "CRITICAL - $1 is ${health_status}."
              exit 2
              ;;
          esac
        else
          /bin/echo "UNKNOWN - Invalid JSON response."
          exit 3
        fi
      }

      # Check if script-parameter is correct.
      if [ "$1" = "lifecycle" ]; then
        health_check_func lifecycle
      elif [ "$1" = "lb_status" ]; then
        health_check_func lb_status
      elif [ "$1" = "is_processing" ]; then
        health_check_func is_processing
      else
        /bin/echo "UNKNOWN - Script parameter is invalid!"
        exit 3
      fi

```