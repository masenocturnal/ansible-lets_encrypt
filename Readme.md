# Let's encrypt with Ansible from scratch 
* Install ansible

    ```$ apt install ansible```

* create dir for playbook

    ```$ mkdir playbooks && cd playbooks```

* Create inventory file. inventory file contains a list of the hosts, host groups or masks that you want to admin.

    ```$ vi inventory ```

        [webservers]

        web01.nocturnal.net.au
    
    *This assumes you have SSH access to the entry. I am using a ~/.ssh/config entry with PKI Certificate Auth

    ** Requires Python to be installed
* Create playbook

    ``` $ vi enable-le-cert.yml```

    ```
    ---
    - hosts: webservers

        tasks:
         - name: "Test connectivity"
           ping:
    ```

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


* Create the role for Let's Encrypt Certificates (you can call it what you like..I lack creativity). This just helps us re-use the role 

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
    
    ```$vi enable-le-cert.yml```

    ```
    ---
    - hosts: webservers

      roles:
        - certificates
    ```

* Create the tasks: Checkout the modules (http://docs.ansible.com/ansible/latest/list_of_all_modules.html)

    ```$ vi ./tasks/main.yml```

In this instance I actually want to run all of these commands locally rather than on the remote server. To accomplish this I use the ```delegate_to:``` feature.

First we need to install the applications and bindings we need to use. Let's encrypt (letsencrypt.org) has an Ubuntu repository (PPA) which we can install signed binaries from. In order to use this, we need to ensure that software-properties-common is installed. We also need the OpenSSL Python bindings to work with OpenSSL directly. 

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

Because we *do* want to prompt for a password, we need to add the -K flag to the ansible-playbook command to get it to prompt. 

```ansible-playbook -K -i inventory enable-le-cert.yml ```

- Create  directory for the Certificates to live

        - name: "Where are we shoving the certs?"
          file:
            path: "~/le-certs"
            state: directory
            mode: 0700
          delegate_to: localhost

 - Check for the existence of the private key, if it doesn't exist create it

        - name: "Do we have a private key?"
          stat: 
            path: "~/le-certs/priv_key.pem"
          register: file_exists
          delegate_to: localhost
        
        - name: "We totes need a private key"
          openssl_privatekey:
            path: "~/le-certs/priv_key.pem"
            state: present
          delegate_to: localhost
          when: file_exists.stat.exists == False
    

* DRY principles and laziness dicate that the key path should be a variable. You can automatically expose variables by defining them in the ./vars/main.yml

    ```$ vi ./vars/main.yml```

        ---
        # vars file for certificates
        certs_dir: "~/le-certs"
        private_key: "{{ certs_dir }}/priv_key.pem"

    And then reference them from inside your tasks.

    ```$ vi ./tasks/main.yml```

        ---
        # tasks file for certificates

        - name: "Where are we shoving the certs?"
        file:
            path: "~/le-certs"
            state: directory
            mode: 0700
        delegate_to: localhost

        - name: "Do we have a private key?"
        stat: 
            path: "{{ private_key }}"
        register: file_exists
        delegate_to: localhost

        - name: "We totes need a private key"
        openssl_privatekey:
            path: "{{ private_key }}"
            state: present
        delegate_to: localhost
        when: file_exists.stat.exists == False