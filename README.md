# How to use this repo to install a Shibboleth Identity Provider

## USE AT YOUR OWN RISK!!!
You can use this Ansible playbook to install a Shibboleth IdP according to the [DFN-AAI online documentation](https://doku.tid.dfn.de/de:shibidp:uebersicht). However, you do this at your own risk!

**Please be aware of the fact that the DFN-AAI team is not responsible for any issues caused by this playbook. You still have to understand what your IdP does and you have to take care of securing your server as needed.**

## Contact and Contributions

You are welcome to send feedback about this work to hotline@aai.dfn.de. That said, we cannot adapt this playbook to all possible server setups or support you in details of getting it up and running. Please consult the [Ansible documentation](https://docs.ansible.com/ansible/latest/index.html) for this.
To submit a merge request:

* Log in to https://jugit.fz-juelich.de/ with your institutional login to create your account.
* Drop us an e-mail to hotline@aai.dfn.de to get invited.
* Please make your changes in a branch and open a merge request.

## Prerequisites

* Use a Debian 11 or an Ubuntu 20.04 machine, other distros have not been tested.
* This playbook creates a local MariaDB database for your IdP. However it does not secure your database server. You might want to prepare MariaDB beforehand, e.g. by running
```
apt install mariadb-server mariadb-client libmariadb-java
mysql_secure_installation
```
* To use an existing remote database server instead, change the according settings in `inventory/group_vars/all/vars`. There you can specify if the database shall be created for you or not, too.
* Install Ansible on the machine you want to run the playbook from.
* Get a webserver certificate (see next section for details).
* Get a copy of the CA certificate of your identity management system (see next section for details).

## Certificates
* Installed with this playbook, your IdP will use **different certificates for the webserver and for SAML communication**.
* For **SAML** communication this playbook will generate a selfsigned certificate with a validity of three years for you. It is then inserted into the xml file that you use to upload the IdP to the DFN-AAI metadata administration tool (`metadata/idp-metadata.xml`).
* As to the **webserver**, you have to provide a certificate yourself. To fetch GÉANT TCS certificates via ACME for the webserver, you might want to extend this playbook. Please see the [TCS FAQ](https://doku.tid.dfn.de/de:dfnpki:tcsfaq#acme1) for details. You can also fetch a certificate manually and place it onto the server.
* To use the Apache configuration template for the virtual host, edit the file `roles/shibboleth_idp/vars/group_vars/all/vars`: Specify the absolute paths to your certificate chain and key files in these two variables: `idp_webserver_fullchain` and `idp_webserver_privatekey`. The virtual host config will be written for you, but you'll have to enable it yourself (see below).
* Your IdP has to trust the CA certificate of your identity management system. Copy the CA certificate in `PEM` format into the `root_certificates` folder and rename it to `idm-root-ca.crt`. It will be installed on the server and added to the LDAP and Apache configuration files automatically.

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
* Add two files named logo.png and favicon.ico to the folder `roles/shibboleth-idp/files`.
* Run it! The whole playbook consists of different roles, tasks and subtasks that you can run independently. Refer to the group (test oder production in this case) with the ''-l'' parameter.
  * To install and configure Tomcat, Apache and Shibboleth IdP in one go run. After that, check and enable the virtual host configuration:
  ```sh
  ansible-playbook -i inventory/hosts site.yml -l test --tags shibboleth_idp
  apache2ctl -t
  a2ensite YOUR-FQDN.conf
  systemctl reload apache2
  ```
  * To run an individual role:
  ```sh
  ansible-playbook -i inventory/hosts site.yml -l test --tags tomcat
  ```
  * You can (re-)run any of the subtasks in the shibboleth-idp role with a tag (see `roles/shibboleth_idp/tasks/main.yml`), e.g. the LDAP configuration:
  ```sh
  ansible-playbook -i inventory/hosts site.yml -l test --tags config-ldap
  ```
  * To roll out changes in the templates attribute-resolver.xml or attribute-filter.xml:
  ```sh
  ansible-playbook -i inventory/hosts site.yml -l test --tags config-attributes
  ```
## Testing the new IdP
* Verify the IdP status page: https://YOUR-FQDN/idp/status
* Log in to [DFN-AAI metadata administration](https://mdv.aai.dfn.de) and create a new IdP by uploading the content of `/opt/shibboleth-idp/metadata/idp-metadata.xml` to the administration tool. Check if it looks good and save the form.
* Add the IdP to DFN-AAI-Test, wait 90 minutes, then try to log in for testsp3.aai.dfn.de.

## Pitfalls
* If you delete the autogenerated SAML certificate, a new one will be created upon next run. Please be aware that your have to publish this certificate in our metadata administration tool and that every change needs 60 to 90 minutes to propagate.
* messages.properties have not been adapted.
* The Login and Logout pages have not been adapted.
* No cronjobs are added. Please think of backing up user consent information from `logs/idp-consent-audit.log` for [GDPR compliance](https://doku.tid.dfn.de/de:shibidp:config-consent-dsgvo)!
* Private IP addresses may cause issues in the webserver configuration. Adapt the according template to your needs.
* If you haven't activated local metadata for your institution in our metadata administration tool or if your local metadata file is still empty add a comment to the line `metadata_local:` in `inventory/group_vars/all/vars`. Otherwise you will get a faulty `/opt/shibboleth-idp/conf/metadata-providers.xml` file in your IdP.
