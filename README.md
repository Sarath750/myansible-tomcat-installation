# myansible-tomcat-installation

ansible setup on both linux and ubuntu
--------------------------------------
yum - to install packages on linux
apt - to install packages on ubuntu

Master machine
--------------
ansible -m setup nodes

vi /etc/ansible/hosts
[nodes]
<linux public ip>
<ubuntu public ip>

vi playbook.yml
---------------
---
- hosts : nodes
  tasks : 
  - name : print OS family
    debug : 
      msg : "{{ ansible_os_family }}"

ansible-playbook playbook.yml --syntax-check
ansible-playbook playbook.yml 

    "msg": "RedHat"   --> linux
    "msg": "Debian"   --> ubuntu

debug module used to print the values.

vi installation.yml
-------------------
---
- hosts : nodes
  tasks :
  - name : check OS
    debug :
      msg : "{{ ansible_os_family }}"
  - name : install java on linux machine
    yum :
      name : java-1.8.0-openjdk
      state : present
    when : ansible_os_family == "Redhat"
  - name : install java on ubuntu machine
    apt :
      name : openjdk-8-jre-headless
      state : present
      update_cache : yes
    when : ansible_os_family == "Debian"

java installed in both linux and ubuntu machines

Tomcat pre-requisite --> java
check jinja templates

Tomcat setup on linux machine with ansible playbook
---------------------------------------------------
 vi tomcat.yml
--------------
---
- hosts: nodes
  tasks:
# yum command will list out the installed java and that will be stored in register.
  - name: check java installation
    yum:
      list: java-1.8.0-openjdk
    register: check_java_existance
# debug module will print out the installed java which is stored in register.
  - name: check java installation
    debug:
      msg: "{{ check_java_existance }}"
  - name: Install java if not installed
    yum:
      name: java-1.8.0-openjdk
      state: present
    when: check_java_existance.results | selectattr("yumstate","match","installed") | list | length == 0
# get_url will download the softwares/packages
  - name: download tomcat url
    get_url:
      url: https://dlcdn.apache.org/tomcat/tomcat-10/v10.0.23/bin/apache-tomcat-10.0.23.tar.gz
      dest: /opt
# unarchive module will extract the tar file of apache-tomcat
  - name: extract tomcat
    unarchive:
      src: /opt/apache-tomcat-10.0.23.tar.gz
      dest: /opt
      remote_src: True
# user module will create a user
  - name: Create Tomcat user and Group
    user:
      name: tomcat
      state: present
      comment: "user for comment"
# file module will change ownership of the apache tomcat
  - name: Change directory tomcat permissions
    file:
      path: /opt/apache-tomcat-10.0.23
      state: directory
      recurse: yes
      owner: tomcat
      group: tomcat
      mode: '0755'
# template module will move tomcat service file to node machine and it can be used again
  - name: tomcat service file template
    template:
      src: tomcat-service.yml
      dest: ./tomcat-service
  - name: Copy Tomcat service from local to remote
    copy:
      src: ./tomcat-service
      dest: /etc/systemd/system/
      mode: 0755
      remote_src: yes
  - name: just force systemd to reread configs (2.4 and above)
    systemd:
      daemon_reload: yes
  - name: enable tomcat startup
    service:
      name: tomcat
      state: restarted


vi tomcat-service.yml
---------------------
[Unit]
Description=Apache apache-tomcat-10.0.23 Web Application Container
After=network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.332.b09-1.amzn2.0.2.x86_64/jre
Environment=CATALINA_PID=/opt/apache-tomcat-10.0.23/temp/apache-tomcat-10.0.23.pid
Environment=CATALINA_HOME=/opt/apache-tomcat-10.0.23
Environment=CATALINA_BASE=/opt/apache-tomcat-10.0.23
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'
ExecStart=/opt/apache-tomcat-10.0.23/bin/startup.sh
ExecStop=/bin/kill -15 $MAINPID

User=tomcat
Group=tomcat

[Install]
WantedBy=multi-user.target






Declaring and Reusing variables
-------------------------------
vi tomcat-variables
-------------------
---
# localhost will download all files in master machine

- hosts: localhost
  vars:
    java_version: java-1.8.0-openjdk
    tomcat_download_url: https://dlcdn.apache.org/tomcat/tomcat-10/v10.0.23/bin/apache-tomcat-10.0.23.tar.gz
    tomcat_download_directory: /opt
    tomcat_destination_path: apache-tomcat-10.0.23

  tasks:
# yum command will list out the installed java and that will be stored in register.
  - name: check java installation
    yum:
      list: "{{ java_version }}"
    register: check_java_existance
# debug module will print out the installed java which is stored in register.
  - name: check java installation
    debug:
      msg: "{{ check_java_existance }}"
  - name: Install java if not installed
    yum:
      name: "{{ java_version }}"
      state: present
    when: check_java_existance.results | selectattr("yumstate","match","installed") | list | length == 0
# get_url will download the softwares/packages
  - name: download tomcat url
    get_url:
      url: "{{ tomcat_download_url }}"
      dest: "{{ tomcat_download_directory }}"
# unarchive module will extract the tar file of apache-tomcat
  - name: extract tomcat
    unarchive:
      src: "{{ tomcat_download_directory }}/{{ tomcat_destination_path }}.tar.gz"
      dest: "{{ tomcat_download_directory }}"
      remote_src: True
# user module will create a user
  - name: Create Tomcat user and Group
    user:
      name: tomcat
      state: present
      comment: "user for comment"
# file module will change ownership of the apache tomcat
  - name: Change directory tomcat permissions
    file:
      path: "{{ tomcat_download_directory }}/{{ tomcat_destination_path }}"
      state: directory
      recurse: yes
      owner: tomcat
      group: tomcat
      mode: '0755'
# template module will move tomcat service file to node machine and it can be used again
  - name: tomcat service file template
    template:
      src: tomcat-service.yml
      dest: ./tomcat-service
  - name: Copy Tomcat service from local to remote
    copy:
      src: ./tomcat-service
      dest: /etc/systemd/system/
      mode: 0755
      remote_src: yes
  - name: just force systemd to reread configs (2.4 and above)
    systemd:
      daemon_reload: yes
  - name: enable tomcat startup
    service:
      name: tomcat
      state: restarted


vi tomcat-variables-service.yml
---------------------
[Unit]
Description=Apache apache-tomcat-10.0.23 Web Application Container
After=network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.332.b09-1.amzn2.0.2.x86_64/jre
Environment=CATALINA_PID={{ tomcat_download_directory }}/{{  tomcat_destination_path }}/temp/{{  tomcat_destination_path }}.pid
Environment=CATALINA_HOME={{ tomcat_download_directory }}/{{ tomcat_destination_path }}
Environment=CATALINA_BASE"{{ tomcat_download_directory }}/{{ tomcat_destination_path }}
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'
ExecStart={{ tomcat_download_directory }}/{{ tomcat_destination_path }}/bin/startup.sh
ExecStop=/bin/kill -15 $MAINPID

User=tomcat
Group=tomcat

[Install]
WantedBy=multi-user.target

ansible-playbook -i hostfile terraform-variables



Tag in ansible
--------------
vi tag-docker.yml
-----------------
---
- hosts: localhost
  tasks: 
  - name: install docker
    yum: 
      name: docker
      state: present
    tags: install

  - name: start docker service
    service: 
      name: docker
      state: started       
    tags: install

  - name: stop docker service
    service: 
      name: docker
      state: stopped
    tags: remove

  - name: remove docker
    yum: 
      name: docker
      state: absent
    tags: remove   


ansible-playbook tag-docker.yml --tag=install
ansible-playbook tag-docker.yml --tag=remove
