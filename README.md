# How to use this repo to install a standalone Shibboleth Identity Provider

## DRAFT!!! ENTWURF!!!
Do not use this in production, yet! Feedback and tests needed!

## Prerequisites

* Use a Debian 10 or an Ubuntu 20.04 machine, other distros have not been tested.
* This playbook creates a MariaDB database for your IdP. However it does not secure your database server. You might want to prepare MariaDB beforehand, e.g. by running
```
apt install mariadb-server mariadb-client libmariadb-java
mysql_secure_installation
```
* Install Ansible on the machine you want to run the playbook from.
* Install the following community collections:

```
ansible-galaxy collection install community.crypto
ansible-galaxy collection install community.mysql
```
* Get a certificate for the webserver, e.g. from DFN-PKI. (For SAML communication this playbook will generate a selfsigned certificate for you.)

## Getting started
* Clone this repository.
* Check if the settings in ansible.cfg apply to your setup, e.g. the become_method.
* Add a file called vault-password.txt and add it to your .gitignore file to NEVER upload it to any repository.
```
touch vault-password.txt
echo "vault-password.txt" >> .gitignore
```
* Edit the file and set a vault password.
* Add the FQDN of your server to inventory/hosts, either in the group [test] which is recommended to get started or in the group [production]
* Fill in all variables in the following vars files. Do not edit the lines with references to vault files - those variables will be kept in encrypted files.
  * inventory/group_vars/all/vars: variables that apply to both, a test IdP and a production IdP
  * inventory/group_vars/test/vars: variables that apply to your test IdP only
  * inventory/group_vars/production/vars: variables that apply to your production IdP only
* Create three vault files for sensitive information with the 'ansible-vault create' command. The variables to define in the vault files can be viewed in the example files.
  * inventory/group_vars/all/vault
  * inventory/group_vars/test/vault
  * inventory/group_vars/production/vault
* Add two files named logo.png and favicon.ico to the folder ```roles/shibboleth-idp/files```.
* Run it! The whole playbook consists of different roles, tasks and subtasks that you can run independently. Refer to the group (test oder production in this case) with the ''-l'' parameter.
  * To install and configure Tomcat, Apache and Shibboleth IdP in one go run ```ansible-playbook -i inventory/hosts site.yml -l test --tags shibboleth-idp```. The role shibboleth-idp will automatically run the tomcat and apache roles.
  * To run an individual role: ```ansible-playbook -i inventory/hosts site.yml -l test --tags tomcat```
  * To (re-)run parts of the subtasks in the shibboleth-idp role, e.g. the LDAP configuration: ```ansible-playbook -i inventory/hosts site.yml -l test --tags config-ldap```
  * To change attribute-resolver.xml or attribute-filter.xml: ```ansible-playbook -i inventory/hosts site.yml -l test --tags config-attributes```

## Testing the new IdP
* Verify the IdP status page: https://YOUR-FQDN/idp/status
* Go to DFN-AAI metadata administration, create a new IdP and enter the EntityID (https://YOURFQDN/idp/shibboleth) to query all metadata from the system. Check if it looks good and save the form. There is no need to change the certificate. Ansible generated a SAML certificate for your IdP and it was imported to the metadata administration in the first go.
* Add the IdP the DFN-AAI-Test, wait 90 minutes, then try to login for testsp3.aai.dfn.de.

## Pitfalls
* If you delete the autogenerated SAML certificate, a new one will be created upon next run. Please be aware that the DFN-AAI team has to process selfsigned certificates manually at the moment! **Submit only the certificate you really want to use** and don't lose it afterwards!
* messages.properties have not been adapted.
* The Login and Logout pages have not been adapted.
* No cronjob are added - backup of user consent information is not included.
* Private IP addresses may cause issues in the webserver configuration. Adapt the according template to your needs.
