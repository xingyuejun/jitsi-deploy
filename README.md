# The SWITCH scalable Jitsi Installation

22.3.2020

SWITCH is the Swiss National Research and Network (NREN) and provides various IT services
to Swiss higher education institutions. This ansible playbook is used to set up
a cluster of individual Jitsi Frontend servers (one for each insitiution) that share a 
pool of videobridges. This is a brand new service that was started during the Covid19 crisis
in March 2020. A service description can be found at https://switch.ch/meet

This repository is provided **AS IS** under a MIT license without any implicit or explicit guarantees
that it will work for you.

The setup is geared towards our needs and will most likely not work out of the box for you.

Things to look out for:

* We provision VMs on our internal IaaS cloud SWITChengines (https://switch.ch/engines) using the 
  `build*servers` playbooks. You will need to adapt that to your environment
* We use an internal ansible role (`users`) that provisions our admin users onto our VMs. You will
  need to roll your own. For now, remove the `users` role entry in `provision.yml`
* We use Ubuntu 18.04 as a base operating system. No other configurations have been or will be tested
* We use Shibboleth Authentication. If you are not part of the universities, chances are you will not
  need to know how that works. Remove the `shib` role entry in `provision.yml`
* We use https://site24x7.com for monitoring. The playbook sets up automatic server monitoring. Again,
  remove the `site24x7` role from `provision.yml`
  
Create your own directories for `group_vars`, `hosts_vars` and `inventory` 
Or use the ansible-galaxy approach, then use

    $ cd ansible
    $ ansible-galaxy install -fr roles/requirements.yml

  
## Adding a new service VM (this is very SWITCH specific)

The building process was automated while deploying the different instances. Accordingly, not all `host_vars` contain the necessary information.

To install a new host, you can copy the `host_vars/template.meet.example.com` folder. 
Initially, you only have to insert the VM variables. Just like `jitsi-ORGANISATION`...

Source the corresponding project using `openrc.sample` as a guidance.

    $ ansible-playbook build_jitis_server.yml --extra-vars=@./host_vars/ORGANISATION.meet.switch.ch/vars.yml -D --check

Note: Script might break sometimes because of API call timeouts. (state 2020-03-14)

Then please do: (IP information is displayed after the build execution.)
* Note floating IP in `inventory/production`
* note the private ip (`10.0.x.y`)
* ask for DNS entry `name-of-uni.meet.switch.ch` with IPv4 & IPv6 info
* Create a new folder in `host_vars` with the name of the server (`name-of-uni.meet.switch.ch`)
* Request a PKI certificate for the server
* Create / copy a `vars.yaml` file and enter the following information:
  * private IP (from above)
  * public IPv4 (from above)
  * Hostname
  * Shibboleth Entity (same as Hostname)
  * WAYF - use https://www.switch.ch/aai/participants/allhomeorgs-expert/ to find it
  * the certificate of the NGINX Server
* Create a vault file: `ansible-vault create host_vars/name-of-uni.meet.switch.ch/vault.yml`
* Edit the vault and add
  * vault_callstats_io_secret: ''
  * vault_nginx_ssl_key: |
  * paste the private key
* run the playbook `ansible-playbook -i inventory/production main.yml --limit name-of-uni.meet.meet.switch.ch` 
* login to the server and get the fingerprint of the AAI Shib certificate and `/etc/shibboleth/sp-cert.pem`  
  `openssl x509 -in /etc/shibboleth/sp-cert.pem -fingerprint -sha1 -noout`
* copy the `sp-cert.pem` and `sp-key.pem` to your local machine and add them to the host_var and the vault respectively
* create the RR request at https://rr.aai.switch.ch
  * Name: hostname
  * Entity ID: hostname
  * Home Org: SWITCH
  * Description: "VideoConf service for UNI provided by SWITCH"
  * For support contacts:
    * FirstName: support
    * LastName: support
    * mail: support@switch.ch
  * Attributes:
    * Required: mail, firstname, givename, uniqueid
  * Audience:
    * Limit to the requesting organisation
    * exclude the EduID
  * Paste the Fingerprint into the comment field at the end    
* wait


## Ansible

To provision a server, enter it's IP address in the `inventory/production` file with its hostname.

The run `ansible-playbook -i inventory/production main.yml --limit new_host.meet.example.com` to run the full
installation.

You can limit the playbook runs to specific tasks with the following tags:

* `conf` - only deploy configuration changes (and restart services where necessary)
* `webconf` - only deploy the web config of jitsi-meet (no service disruption)
* `nginx` - only install/configure Nginx
* `jitsi` - only install/configure Jitsi
* `shib` - only install/configure Shibboleth

Example:

    $ ansible-playbook -i inventory/production main.yml --limit new.host.meet.ch --tags conf

## Add a new videobridge server

Source credentials of the `videobridges.meet.switch.ch` project!

* Add new entry in `inventory/production` in the section `videobridge`.
* Run the following command to build (Temporarily comment out the existing hosts in `inventory/production [videobridge]` otherwise the script will block the creation of the new servers!)

    $ ansible-playbook build_videobridge_servers.yml --extra-vars=@./host_vars/videobridges.meet.switch.ch/vars.yml -D

* Write the IPs in `inventory/production` in the section `videobridge`
* Assure that you filled in the `callstats.io` credentials
* Execute the following playbook:

    $ ansible-playbook -i inventory/production main.yml --limit ORGANISATION.meet.example.com
    

## Add jibri service
There is a build script which deploys new jibri servers. **Please comment out the existing servers!**
Source the project's `.openrc` file and run:

    $ ansible-playbook -i inventory/test build_jibri_server.yml -DC

This script loops over the `jibri` group and creates new hosts in the given project.

## Purge Services

While installing and trying to get things working, you might want to purge all traces of
*nginx* or *jitsi* from your servers. Use the `purge.yml` playbook like so:

    $ ansible-playbook -i inventory/production purge.yml --limit ORGANISATION.meet.example.com

Specify one (or more of the following tags) with `--tags ...`
  * `nginx` - remove nginx from server
  * `jitsi` - remove all traces of jitsi / prosody etc from the server
  * `jicofolog` - truncate the jicofo logs

## Attribution

The Jitsi Ansible role is heavily influenced by https://github.com/freedomofpress/ansible-role-jitsi-meet

## License
MIT Licence

Copyright 2020 SWITCH, https://switch.ch

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
