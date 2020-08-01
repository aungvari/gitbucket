Detailed information about the task
===============

## Prepare the firewall
  - Install and enable firewalld: this was already done according to `systemctl status firewalld.service`
  - Create a new firewall zone called "coconut" with the source 17.3.0.32/32: Created a zone xml file which can be found in this repository `coconut-zone.xml`
  - Create a new firewall service called "coconut" with the 54823/tcp and 32833/udp ports: Created a service xml file which can be found in this repository `coconut-service.xml`
  - Enable the "coconut" service for the "coconut" zone: Should be working after `firewall-cmd --reload` and the xml files under /etc/firewalld/zones and /etc/firewalld/service, alternatively we can use `firewall-cmd --zone=coconut --new-service-from-file=coconut.xml --permanent`
  - Make sure these changes persist after restarts. 
  - Had to add the DNS service to the public zone to be able to reach repositories with `firewall-cmd --zone=public --add-service=dns --permanent`
  - Check settings with `firewall-cmd --get-active-zones` and `firewall-cmd --list-services --zone=coconut`
  - Sources used: https://firewalld.org/documentation/man-pages/firewalld.zone.html and https://firewalld.org/documentation/man-pages/firewalld.service.html

## Install tomcat/gitbucket and postgresql
  - Use the latest release from GitHub. Serve it in any servlet container you like: Since Tomcat is removed from Centos8, I had to use and a third party repository: https://centos.pkgs.org/8/harbottle-main-x86_64/tomcat9-9.0.37-1.el8.harbottle.x86_64.rpm.html and follow the installation instructions then copy gitbucket.war to the webapps folder
  - Make it start automatically upon boot as a systemd service: `systemctl enable tomcat9.service`
  - Serve the webapp on port 32887 (TCP) in plain HTTP, and pass it through the firewall in the previously created "coconut" zone via a new "gitbucket" firewall service: In /etc/tomcat9/server.xml change the `Connector port="8080"` to "32887" and add the following in the <Host> context: 
  ```
        <Context path="" docBase="gitbucket">
        <WatchedResource>WEB-INF/web.xml</WatchedResource>
        </Context>
  ```
  Note: normally this should be added in a seperate file. Gitbucket service.xml is added in this repository. To assign it to the coconut zone permanently: `firewall-cmd --zone=coconut --add-service=gitbucket --permanent`
  - Make sure SELinux remains in enforcing mode. Ensure that no GitBucket actions are denied by SELinux (create the necessary adjustments if needed): had to use this fix https://github.com/gitbucket/gitbucket/tree/master/contrib/linux/redhat/selinux and adjusting the directories (se_fix.sh) in addition to the steps
  ```
  chown root.tomcat /var/lib/tomcat9/webapps/gitbucket.war
  restorecon -v /var/lib/tomcat9/webapps/gitbucket.war
  ln -s /usr/share/tomcat9/.gitbucket /opt/gitbucket
  ```
  - Optional: install PostgreSQL and make GitBucket use that instead of the default H2 database: `yum install postgresql-server` then run `postgresql-setup initdb` and the create the gitbucket user and database as user postgres
  ```
  createuser --username=postgres --no-createrole --no-createdb --no-inherit --encrypted --login --pwprompt --no-superuser gitbucket
  createdb --encoding=UTF-8 --owner=gitbucket gitbucket
  ```
  Finally, change the connection string in `/opt/gitbucket/database.conf` according to https://github.com/gitbucket/gitbucket/wiki/External-database-configuration#postgresql-95-or-higher and make postgresql start up on boot with `systemctl enable postgresql.service`
  - Optional: serve GitBucket in the root (http://46.101.227.60:32887/) instead of the /gitbucket subdir: Done with the Context path in server.xml
  - Optional: run the webapp under a restricted user instead of root/root: This was achieved by using the third party Tomcat RPM, no extra steps were needed, can be configured in `/lib/systemd/system/tomcat9.service`

## Configure gitbucket

  - Change the root user's password to "IDDQD": changed on the UI under "Your profile" when logged in as root
  - Create the "aimotive" normal (non-admin) user with the "usetheforce" password: created on the UI
  - Create a private Git repository named "goodstuff", and make this "aimotive" user the owner of it: logged in as aimotive and created the repository on the UI
  - Optional: make a git commit in this new repo with any text file you like.
