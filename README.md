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
* This playbook creates a MariaDB database for your IdP. However it does not secure your database server. You might want to prepare MariaDB beforehand, e.g. by running
```
apt install mariadb-server mariadb-client libmariadb-java
mysql_secure_installation
```
* Install Ansible on the machine you want to run the playbook from.

## Certificates
* Installed with this playbook, your IdP will use **different certificates for the webserver and for SAML communication**.
* For SAML communication this playbook will generate a selfsigned certificate with a validity of three years for you.
* For the webserver certificate, the playbook generates a private key and a CSR for you on first run. **THUS, IT WILL "FAIL" ON FIRST RUN! This is expected as you need your certificate authority to sign the CSR it generates for you.** Save the certificate in the directory you cloned the repository into. Rename the file to reflect you IdP and the server's FQDN like so: IDPVIRTUALHOST_MACHINEFQDN.crt.pem, e.g. idp.example.org_machine.some.fqdn.crt.pem. Run the playbook again. The rationale behind this procedure is to keep the private key only on the machine that needs it.
* If you insist on providing your own existing private key and certificate, save both files in the local folder you cloned the repository into. Rename them to IDPVIRTUALHOST_MACHINEFQDN.crt.pem and IDPVIRTUALHOST_MACHINEFQDN.key.pem, e.g. idp.example.org_machine.some.fqdn.crt.pem and idp.example.org_machine.some.fqdn.key.pem and add `--tags=all,shibboleth_own_key` to your ansible invocation. Note that it is not recommended to keep your private key in that location!
* If you are planning to fetch GÉANT TCS certificates via ACME for the webserver, you have to extend this playbook yourself. Please see the [TCS FAQ](https://doku.tid.dfn.de/de:dfnpki:tcsfaq#acme1) for details.

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
  * To install and configure Tomcat, Apache and Shibboleth IdP in one go run
  ```sh
  ansible-playbook -i inventory/hosts site.yml -l test --tags shibboleth_idp
  ```
  The role shibboleth-idp will automatically run the tomcat and apache roles.
  * To run an individual role:
  ```sh
  ansible-playbook -i inventory/hosts site.yml -l test --tags tomcat
  ```
  * To (re-)run parts of the subtasks in the shibboleth-idp role, e.g. the LDAP configuration:
  ```sh
  ansible-playbook -i inventory/hosts site.yml -l test --tags config-ldap
  ```
  * To change attribute-resolver.xml or attribute-filter.xml:
  ```sh
  ansible-playbook -i inventory/hosts site.yml -l test --tags config-attributes
  ```

For development purposes you can create a local certificate authority as follows:
```sh
ansible-playbook setup_dev.yml
```

Signing your CSR is then done as follows:
```sh
ansible-playbook selfsign_csr.yml
```

## Testing the new IdP
* Verify the IdP status page: https://YOUR-FQDN/idp/status
* Go to DFN-AAI metadata administration, create a new IdP and enter the EntityID (https://YOURFQDN/idp/shibboleth) to query all metadata from the system. Check if it looks good and save the form. There is no need to change the certificate. Ansible generated a SAML certificate for your IdP and it was imported to the metadata administration in the first go.
* Add the IdP the DFN-AAI-Test, wait 90 minutes, then try to login for testsp3.aai.dfn.de.

## Pitfalls
* If you delete the autogenerated SAML certificate, a new one will be created upon next run. Please be aware that your have to publich this certificate in our metadata administration tool and that every change needs 60 to 90 minutes to propagate.
* messages.properties have not been adapted.
* The Login and Logout pages have not been adapted.
* No cronjob are added - backup of user consent information is not included.
* Private IP addresses may cause issues in the webserver configuration. Adapt the according template to your needs.
* If you haven't activated local metadata for your institution in our metadata administration tool or if your local metadata file is still empty add a comment to the line `metadata_local:` in `inventory/group_vars/all/vars`. Otherwise you will get a faulty `/opt/shibboleth-idp/conf/metadata-providers.xml` file in your IdP.
