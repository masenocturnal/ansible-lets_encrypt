# Let's encrypt with Ansible from scratch 

This walkthrough is an attempt to document the process of using Ansible (without AWX / Tower) in order to create, submit a Let's Encrypt certificate with multiple SANs on the Ansible hosts and deploy the resulting certificate to any number of webservers. 

## Assumptions

* You have an Ubuntu 16.04 (or similar) Ansible host. For other distributions (RHEL), *BSD etc.. the process is basically the same, but the you'll use the native package manager for your distro. For those of you on Windows this should work on WLS if you have a newer version of Windows 10. 

* You have an existing web server which is configured correctly for the Well-Known URI's. [RFC 5785](https://tools.ietf.org/html/rfc5785 )

## Background 

**__Ansible playbooks are designed to be Idempotent__** 

You should be able to run a playbook indefiniately and the system you are configuring should be in the same state every time. Ansible will perform exactly the tasks you specify and nothing more. However, if Ansible detects that the action has already been performed, it will skip the task and move on to the next one. 

Unfortunately or fortunately, depending on how you look at it, Let's Encrypt have hard limits on the number of requests you can perform a day. We are assuming that you don't intend to exceed this limit. The playbook can be more robust, but for this demonstration I'm assuming restraint.

## Process

* Install ansible from the Ansible PPA (http://docs.ansible.com/ansible/latest/intro_installation.html)

    ```
        $ sudo apt-add-repository ppa:ansible/ansible
        $ sudo apt-get update
        $ apt install ansible
    ```

* create a directory for the playbook

    ```$ mkdir playbooks && cd playbooks```

* Create inventory file. The inventory file contains a list of the hosts, host groups or masks that you want to admin.

    In this instance we are defining a list of webservers which we will run the playbook against.

    ```$ vi inventory ```

        [webservers]

        web01.nocturnal.net.au
    
    *This assumes you have SSH access to the entry. I am using a ~/.ssh/config entry with PKI Certificate Auth

    ** Requires Python to be installed
* Create the playbook it's self

    ``` $ vi enable-le-cert.yml```

    ```
    ---
    - hosts: webservers

        tasks:
         - name: "Test connectivity"
           ping:
    ```

    We can use the **name** directive in order to give our playbooks some contextual information, e.g what are we doing? Why are we doing it ? These get printed out when the tasks are run and when written with thought allow anyone to make sense of what your playbooks are doing and more importantly why. 

* Check that it executes successfully.

    ```$ ansible-playbook -i inventory enable-le-cert.yml```

    ```
    am@seattle:~/letsencrypt/playbooks$ ansible-playbook -i inventory enable-le-cert.yml

    PLAY [webservers] **********************************************************************************************************************

    TASK [Gathering Facts] *****************************************************************************************************************
    ok: [web01.nocturnal.net.au]

    TASK [Test connectivity] ***************************************************************************************************************
    ok: [web01.nocturnal.net.au]

    PLAY RECAP *****************************************************************************************************************************
    web01.nocturnal.net.au     : ok=2    changed=0    unreachable=0    failed=0   

    ```


* Create the role for Let's Encrypt Certificates (you can call it what you like..I lack creativity). This just helps us re-use the tasks we create in other playbooks. 

    ```$ ansible-galaxy init certificates```

    ```
    am@seattle:~/letsencrypt/playbooks$ tree 
    .
    ├── certificates
    │   ├── defaults
    │   │   └── main.yml
    │   ├── files
    │   ├── handlers
    │   │   └── main.yml
    │   ├── meta
    │   │   └── main.yml
    │   ├── README.md
    │   ├── tasks
    │   │   └── main.yml
    │   ├── templates
    │   ├── tests
    │   │   ├── inventory
    │   │   └── test.yml
    │   └── vars
    │       └── main.yml
    ├── enable-le-cert.yml
    ├── inventory
    └── Readme.md

    9 directories, 11 files
    ```

* Modify the playbook to use the roles
    
    ``` $vi enable-le-cert.yml```

    ```
    ---
    - hosts: webservers

      roles:
        - certificates
    ```

    The ```hosts: webservers``` directive tells ansible that we want to run the playbook in against all the __webservers__ listed in our inventory.

* Create the tasks: Checkout the modules (http://docs.ansible.com/ansible/latest/list_of_all_modules.html)

    ```$ vi ./tasks/main.yml```

In this instance I actually want to run all of these commands locally rather than on the remote server. To accomplish this I use the ```delegate_to:``` feature.

First we need to install the applications and bindings we need to use. Let's encrypt (letsencrypt.org) has an Ubuntu repository (PPA) which we can install signed binaries from. In order to use this, we need to ensure that software-properties-common is installed. We also need the OpenSSL Python bindings to work with OpenSSL directly. The requirements are listed in the documentation for each module. 

    - name: "Install the python openssl-bindings and software-properties-common so we install use OpenSSL and certbot"
      apt:
        name: {{ item }} 
        state: installed    
      with_items:
        - python3-openssl
        - software-properties-common
      delegate_to: localhost
      become: True
    
Note the ```become: True```. We need elevated privileges in order to be able to install software.  The *become* directive is ansibles method for doing that. Because I intend to be able to run this from my desktop, this means that we will also want to prompt for the password I would supply to sudo. If this were a server system with a dedicated user, I might have /etc/sudoers set to not prompt when elevating priviledges. 

Because we *do* want to prompt for a password, we need to add the -K flag to the ansible-playbook command to get it to prompt... There are better / less cumbersome ways to deal with this scenario, but that's for another time.

```$ ansible-playbook -K -i inventory enable-le-cert.yml ```

- Create  directory for the Certificates to live

        - name: "Where are we shoving the certs?"
          file:
            path: "~/le-certs"
            state: directory
            mode: 0700
          delegate_to: localhost

 - Let's Encrypt / the ACME protocol (which Let's Encrypt uses) requires that we have different *account* and *certificate keys*. We need to generate both, but let's look at the procedure for the account key. The certificate key will be the same process

        ... 

        - name: "Do we have an existing account key?"
          stat: 
            path: "~/le-certs/account_key.pem"
          register: file_exists
          delegate_to: localhost

        - name: Create Let's Encrypt account key
          openssl_privatekey:
            path: "~/le-certs/account_key.pem"
            state: present
          delegate_to: localhost
          when: file_exists.stat.exists == False

          ....


    

* DRY principles and laziness dicate that the key path should be a variable. You can automatically expose variables by defining them in the ./vars/main.yml. This is just the tip of the iceberg with regards to using variables within Ansible.

    ```$ vi ./vars/main.yml```

        ---
        # vars file for certificates
        certs_dir: "~/le-certs"
        account_key: "{{ certs_dir }}/account_key.pem"

    And then reference them from inside your tasks.

    ```$ vi ./tasks/main.yml```

        ---
        # tasks file for certificates
        ...

        - name: "Do we have an existing account key?"
          stat: 
            path: "{{ account_key }}"
          register: file_exists
          delegate_to: localhost

        - name: Create Let's Encrypt account key
          openssl_privatekey:
            path: "{{ account_key }}"
            state: present
          delegate_to: localhost
          when: file_exists.stat.exists == False

    Notice how we are also using the ```when``` directive. In this instance we are using the previous ```stat``` module to register a variable **file_exists** and then we are subsequently checking to see that the **exists** property is **False**.
    
    This tells Ansible not to execute the task when an account key has already been created. 

    Here we are using the [stat](http://docs.ansible.com/ansible/latest/stat_module.html) module. The stat module maps to the POSIX syscall that returns attributes of an inode. For those of you on Windows, an inode is similar to a File ID. Basically we are checking to see if the file exists.

## Final Result

Now that we can read and understand individual tasks, now we can look at how they can be combined together to create a living playbook for installing a certificate.



```
---
# tasks file for certificates
- name: Install the python openssl-bindings and software-properties-common so we install use OpenSSL and certbot
  apt:
    name: "{{ item }}"
    state: installed    
  with_items:
    - python3-openssl
    - software-properties-common
  delegate_to: localhost
  become: True

- name: "Create the certificate storage dir?"
  file:
    path: "~/le-certs"
    state: directory
    mode: 0700
  delegate_to: localhost

- name: "Do we have an existing account key?"
  stat: 
    path: "{{ account_key }}"
  register: file_exists
  delegate_to: localhost

- name: Create Let's Encrypt account key
  openssl_privatekey:
      path: "{{ account_key }}"
      state: present
  delegate_to: localhost
  when: file_exists.stat.exists == False

- name: "Do we have a private key?"
  stat: 
    path: "{{ private_key }}"
  register: file_exists
  delegate_to: localhost

- name: "Create a private key for the certificates"
  openssl_privatekey:
      path: "{{ private_key }}"
      state: present
  delegate_to: localhost
  when: file_exists.stat.exists == False

- name: Add the Certbot (let's encrypt ACME client) PPA
  apt_repository:
    repo: 'ppa:certbot/certbot'
    state: present
  become: True
  delegate_to: localhost

- name: Install Certbot
  apt:
    name: certbot
    state: latest
    update_cache: yes
  become: True
  delegate_to: localhost

- name: "Ensure that the well-known directory is present on the server"
  file:
    state: directory
    path: "{{ well_known_path }}/.well-known/acme-challenge"
    mode: 0755
  tags:
    - generate_cert

- name: "Create the CSR"
  openssl_csr:
    path: "{{ csr_path }}"
    privatekey_path: "{{ private_key }}"
    common_name: "{{ primary_domain }}"
    country_name: "{{ crt_country_name }}"
    email_address: "{{ crt_email_address }}"
    locality_name: "{{ crt_locality_name }}"
    organization_name: "{{ crt_organization_name }}"
    subject_alt_name: "{{ additional_domains }}"
  delegate_to: localhost
  tags:
    - generate_cert

- name: "Readback the CSR"
  raw: "openssl req -in {{ csr_path }} -noout -text"
  register: csr
  delegate_to: localhost
  tags:
    - generate_cert

- name: Display CSR
  debug:
    var: csr.stdout_lines
  delegate_to: localhost
  tags:
    - generate_cert

# - name: Test loop
#   debug:
#     msg: "domain is : {{ item[4:] }}"
#   with_items: "{{ additional_domains }}"
#   tags:
#     - generate_cert

- name: "Submit Certificate to Let's Encrypt"
  letsencrypt:
     account_key: "{{ account_key }}"
     csr: "{{ csr_path }}"
     dest: "{{ certs_dir}}/{{ primary_domain }}.crt"
     # acme_directory: "https://acme-v01.api.letsencrypt.org/directory"
     
  delegate_to: localhost    
  register: le_challenge
  tags:
    - generate_cert

- name: "Add challenge/response files to the server. Assumes webserver config is setup for the ACME protocol"
  copy:
     dest: "{{ well_known_path }}/{{item.value['http-01']['resource'] }}"
     content: "{{ item.value['http-01']['resource_value'] }}"
  when: le_challenge|changed
  with_dict: "{{ le_challenge['challenge_data'] }}"
  tags:
    - generate_cert

- name: "Respond to Let's encrypt with challenge data"
  letsencrypt:
    account_key: "{{ account_key }}"
    csr: "{{ csr_path }}"
    dest: "{{ certs_dir }}/{{ primary_domain }}.crt"
    data: "{{ le_challenge }}"
    # acme_directory: "https://acme-v01.api.letsencrypt.org/directory"
    
  delegate_to: localhost  
  tags:
    - generate_cert

- name: "Copy Certificate Private Key to the server"
  copy:
    src: "{{ private_key }}"
    dest: "{{ server_private_key_path }}"
    owner: "{{ server_private_key_owner }}"
    group: "{{ server_private_key_group }}"
    mode: "{{ server_private_key_mode }}"
    force: yes
  become: True
  tags:
    - copy_cert

- name: "Copy Certificate to the server"
  copy:
    src: "{{ certs_dir }}/{{ primary_domain }}.crt"
    dest: "{{ server_crt_path }}"
    owner: "{{ server_crt_owner }}"
    group: "{{ server_crt_group }}"
    mode: "{{ server_crt_mode }}"
    force: yes
  become: True
  tags:
    - copy_cert

# - name: "Reload the webserver"
# Intentionally left blank. 
# You can get Ansible to trigger a reload but as an incorrect certificate will 



```


```
---

# vars file for certificates
primary_domain: "example.com"
additional_domains:
  - "DNS:example.org"
  - "DNS:{{ primary_domain }}" 
  - "DNS:www.example.com"
  - "DNS:www..example.org"
certs_dir: "~/le-certs"
private_key: "{{ certs_dir }}/priv_key.pem"
account_key: "{{ certs_dir }}/account_key.pem"
public_key: "{{ certs_dir }}/pub_key.pem"
csr_path: "{{ certs_dir }}/{{ primary_domain }}.csr"
well_known_path: "/tmp/letsencrypt/public_html/"

server_private_key_path: "/etc/ssl/private/{{ primary_domain }}.key"
server_private_key_owner: "root"
server_private_key_group: "root"
server_private_key_mode: "0600"


server_crt_path: "/etc/ssl/certs/{{ primary_domain }}.pem"
server_crt_owner: "root"
server_crt_group: "root"
server_crt_mode: "0600"


crt_country_name: "AU"
crt_email_address: "domain@example.com"
crt_locality_name: "Adelaide"
crt_organization_name: "My Company"

```

## My certificates' chain doesn't validate!

Correct. In order to avoid making production certs, the folks at Ansible wisely decided to use the test directory by default. 

If you want to mint real live certs, just uncomment the ```acme_directory``` argument in the lets_encrypt task. 

## Where to next?

What we have is ok, but we can't really use it for lots of domains without modifying the vars/main.yml directly. Fortunately Ansible allows us to inject variables into our roles. 

@todo
