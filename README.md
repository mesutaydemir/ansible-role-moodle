# Installation Moodle 4.1.1 Though Ansible
## Prequisites
- Virtualbox installed Ubuntu Linux machine (Preferrably Ubuntu 22.04 Desktop) having at least 16GB Ram, 250GB Disk
- Familiarity with Linux environments and commands

## Vagrant Installation
In our Ubuntu machine (not the control node), run the following commands to instal & check the vagrant version.
```
sudo apt install vagrant
vagrant --v
```
### Preparing Vagrant File
Next, we'll use the following Vagrant file to create control and host machines. All machines will be Ubuntu 22.04 with 1GB Ram, 2 Core CPUs and 40 GB Disk.

```
mkdir ~/Desktop/vagrant
vim Vagrantfile
```
Copy and paste the following contents into this file and then save and exit:
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  config.vm.box_version = "4.2.12"

  config.vm.define "control", primary: true do |control|
    control.vm.hostname = "control"

    control.vm.network "private_network", ip: "192.168.56.10"

    control.vm.provision "shell", inline: <<-SHELL
      apt update > /dev/null 2>&1 && apt install -y python3.8-venv sshpass > /dev/null 2>&1
    SHELL
  end

  (0..2).each do |i|
    config.vm.define "host#{i}" do |node|
      node.vm.hostname = "host#{i}"

      node.vm.network "private_network", ip: "192.168.56.#{i+20}"

      node.vm.provision "shell", inline: <<-SHELL
        sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
        systemctl restart sshd.service
      SHELL
    end
  end
end
```


### Vagrant Komutları
```ruby
vagrant status ==> tüm makinelerin durumunu kontrol etmek için kullanılır.
vagrant box list
vagrant up ==> Vagrant dosyasındaki verilere göre makineleri oluşturur.
vagrant destroy -f ==> tüm makineleri silmek için kullanılır.
vagrant --help
vagrant ssh ==> Varsayılan olarak kontrol makinesine bağlanır.
vagrant box --help
vagrant suspend ==> tüm makineleri uyku moduna alır.
vagrant resume ==> tüm makineleri uyku modundan çıkarıp başlatır.
vagrant init ubuntu/focal64 --box-version 20230119.0.0 ==> vagrant makinesini up etmeden önce oluşturulan vagrant dosyası
```
## Ansible kurulumu
- Öncelikle `vagrant ssh` komutu ile kontrol makinresine bağlantı yapıyoruz
- yazılım reposunu güncelleyip python3 virtual environment paketini *python3-venv* kurmayabiliriz :) çünkü yukarıdaki Vagrant Dosyasında (provisioning file) var:

```
sudo apt update
sudo apt install python3-venv -y
```

- Ev dizinimizin altında merkezi bir dosya altında yapılandırma dosyalarını oluşturuyoruz:
```
python3 -m venv ~/.venv/kamp
```
python3'ün path'ini belirtmek için aşağıdaki komutu çalıştırıyoruz:
```
source ~/.venv/kamp/bin/activate
```

- Bu komutu etkisizleştirmek için: 
```
deactivate
```
`which python3` komutu ile */usr/bin/python3* görülecektir. `source ~/.venv/kamp/bin/activate` komutundan sonra *venv* altındaki python3 
```
pip3 install --upgrade pip # pip3 paketini güncelliyoruz.
```
### Dependancy management
- pythonda kullanılacak kütüphaneleri tanımlamak için *requirements.txt* dosyasını /home/vagrant altında oluşturuyoruz:
```
nano requirements.txt
```
- Dosyanın içine aşağıdaki satrıları ekledikten sonra kaydedip çıkıyoruz:
```ruby
ansible==6.7.0
ansible-core==2.13.7
cffi==1.15.1
cryptography==39.0.0
Jinja2==3.1.2
MarkupSafe==2.1.2
packaging==23.0
pkg_resources==0.0.0
pycparser==2.21
PyYAML==6.0
resolvelib==0.8.1
```
- Aşağıdaki komutlar ile önce pip'i upgrade ediyoruz. Sonra *requirements.txt* içinde belirtilen python kütüphanelerini (versiyonlarında belirtildiği şekilde) güncelliyoruz. 

>Not: Kütüphanelerin versiyonları güvenlik güncellemeleri ve buglardan dolayı ara sıra güncellenerek kontrol edilmeli.

`pip3 freeze` requirements.txt dosyasındaki kütüphaneleri listelemek için

`pip3 install -r requirements.txt` **declarative**
`pip3 install ansible` yazarak da kurabilirdim **imperative**

- hosts dosyasını oluşturuyoruz. Yöneteceğimiz makineleri bu dosyada tanımlıyoruz.
```
nano hosts
```
- Oluşturduğumuz host dosyasının içeriği aşağıdaki gibi:
```ruby
host0 ansible_host=192.168.56.20
host1 ansible_host=192.168.56.21
host2 ansible_host=192.168.56.22

[all:vars]
ansible_user=vagrant
ansible_password=vagrant
```
> NOT! ansible `sudo apt install ansible` komutu ile kurulmuyorsa `hosts`,`ansible.cfg` ve `playbook.yml` dosyaları aynı dizinde olmalı.
## Ad-hoc 
Bu komutlar ile önce bağlantıyı kontrol edeceğiz:

- Kontrol makinesinde yönetilen makinelere bağlantı yapılabildiğini doğrulayalım:

```
ansible all -i hosts -m ping --ssh-common-args='-o StrictHostKeyChecking=no'
```

- Alternatif olarak bu klasörde *ansible.cfg* dosyasını oluşturup aşağıdaki satırları ekledikten sonra:
```
[defaults]
host_key_checking = False
inventory = hosts
```
aşağıdaki komutu çalıştırabiliriz:
```
ansible all -i hosts -m ping 
```
## Ansible Galaxy ile moodle rolünün çekilmesi
> Ansible galaxy koleksiyonlarına https://galaxy.ansible.com adresinden erişilebilir. `ansible-galaxy install buluma.moodle` komutu çalıştırıldığında koleksiyon dosyaları `/home/vagrant/.ansible/roles/buluma.moodle` dizinine kopyalanır. Hnagi komut ve rol adının kullanılacağı koleksiyonda yer alan *Details* ve *ReadMe* sekmelerinden erişilmelidir.

> `ansible-galaxy search paket_adı --author yazar_adi` komutu ile belirli bir yazar tarafından yazılmış ansible-galaxy şablonu aratılabilir.
- Komut sonrası oluşturulan dosya/klasörlerin açıklamalarına (https://danuka-praneeth.medium.com/ansible-roles-and-ansible-galaxy-b224f4693cd4) erişilebilir. Şimdi `ansible-galaxy install buluma.moodle` komutu sonrası oluşan dizin ağacı aşağıdaki gibidir:
```ruby
vagrant@control:~/.ansible/roles/buluma.moodle$ tree
.
├── CHANGELOG.md
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── LICENSE
├── README.md
├── SECURITY.md
├── defaults
│   └── main.yml
├── meta
│   ├── argument_specs.yml
│   ├── main.yml
│   └── preferences.yml
├── molecule
│   └── default
│       ├── ansible.cfg
│       ├── apt_update.yml
│       ├── converge.yml
│       ├── hosts
│       ├── molecule.yml
│       ├── php.yml
│       ├── prepare.yml
│       └── verify.yml
├── requirements.txt
├── requirements.yml
├── tasks
│   ├── assert.yml
│   └── main.yml
├── templates
│   └── config.php.j2
├── tox.ini
└── vars
    └── main.yml

7 directories, 25 files
``` 



Install and configure moodle on your system.


## [Example Playbook](#example-playbook)

This example is taken from `molecule/default/converge.yml` and is tested on each push, pull request and release.
```yaml
---
- name: converge
  hosts: all
  become: yes
  gather_facts: yes

  roles:
    - role: buluma.moodle
```

The machine needs to be prepared. In CI this is done using `molecule/default/prepare.yml`:
```yaml
---
- name: prepare
  hosts: all
  become: yes
  gather_facts: no

  roles:
    - role: buluma.bootstrap
    - role: buluma.buildtools
    - role: buluma.epel
    - role: buluma.mysql
      mysql_databases:
        - name: moodle
          encoding: utf8mb4
          collation: utf8mb4_unicode_ci
      mysql_users:
        - name: moodle
          password: moodle
          priv: "moodle.*:ALL"
    - role: buluma.python_pip
    - role: buluma.openssl
      openssl_items:
        - name: apache-httpd
          common_name: "{{ ansible_fqdn }}"
    - role: buluma.php
    - role: buluma.selinux
    - role: buluma.httpd
      httpd_vhosts:
        - name: moodle
          servername: moodle.example.com
    - role: buluma.cron
    - role: buluma.core_dependencies
```


## [Role Variables](#role-variables)

The default values for the variables are set in `defaults/main.yml`:
```yaml
---
# defaults file for moodle

# The version of moodle to install.
moodle_version: 310

# A path where to save the data.
moodle_data_directory: /opt/moodledata

# The permissions of the created directories.
moodle_directory_mode: "0750"

# Details to connect to the database.
moodle_database_type: mysqli
moodle_database_hostname: localhost
moodle_database_name: moodle
moodle_database_username: moodle
moodle_database_password: moodle
moodle_database_prefix: ""

# The URL where to serve content.
moodle_wwwroot: "https://{{ ansible_default_ipv4.address }}/moodle"
```

## [Requirements](#requirements)

- pip packages listed in [requirements.txt](https://github.com/buluma/ansible-role-moodle/blob/main/requirements.txt).

## [Status of used roles](#status-of-requirements)

The following roles are used to prepare a system. You can prepare your system in another way.

| Requirement | GitHub | GitLab |
|-------------|--------|--------|
|[buluma.bootstrap](https://galaxy.ansible.com/buluma/bootstrap)|[![Build Status GitHub](https://github.com/buluma/ansible-role-bootstrap/workflows/Ansible%20Molecule/badge.svg)](https://github.com/buluma/ansible-role-bootstrap/actions)|[![Build Status GitLab ](https://gitlab.com/buluma/ansible-role-bootstrap/badges/main/pipeline.svg)](https://gitlab.com/buluma/ansible-role-bootstrap)|
|[buluma.buildtools](https://galaxy.ansible.com/buluma/buildtools)|[![Build Status GitHub](https://github.com/buluma/ansible-role-buildtools/workflows/Ansible%20Molecule/badge.svg)](https://github.com/buluma/ansible-role-buildtools/actions)|[![Build Status GitLab ](https://gitlab.com/buluma/ansible-role-buildtools/badges/main/pipeline.svg)](https://gitlab.com/buluma/ansible-role-buildtools)|
|[buluma.cron](https://galaxy.ansible.com/buluma/cron)|[![Build Status GitHub](https://github.com/buluma/ansible-role-cron/workflows/Ansible%20Molecule/badge.svg)](https://github.com/buluma/ansible-role-cron/actions)|[![Build Status GitLab ](https://gitlab.com/buluma/ansible-role-cron/badges/main/pipeline.svg)](https://gitlab.com/buluma/ansible-role-cron)|
|[buluma.core_dependencies](https://galaxy.ansible.com/buluma/core_dependencies)|[![Build Status GitHub](https://github.com/buluma/ansible-role-core_dependencies/workflows/Ansible%20Molecule/badge.svg)](https://github.com/buluma/ansible-role-core_dependencies/actions)|[![Build Status GitLab ](https://gitlab.com/buluma/ansible-role-core_dependencies/badges/main/pipeline.svg)](https://gitlab.com/buluma/ansible-role-core_dependencies)|
|[buluma.epel](https://galaxy.ansible.com/buluma/epel)|[![Build Status GitHub](https://github.com/buluma/ansible-role-epel/workflows/Ansible%20Molecule/badge.svg)](https://github.com/buluma/ansible-role-epel/actions)|[![Build Status GitLab ](https://gitlab.com/buluma/ansible-role-epel/badges/main/pipeline.svg)](https://gitlab.com/buluma/ansible-role-epel)|
|[buluma.httpd](https://galaxy.ansible.com/buluma/httpd)|[![Build Status GitHub](https://github.com/buluma/ansible-role-httpd/workflows/Ansible%20Molecule/badge.svg)](https://github.com/buluma/ansible-role-httpd/actions)|[![Build Status GitLab ](https://gitlab.com/buluma/ansible-role-httpd/badges/main/pipeline.svg)](https://gitlab.com/buluma/ansible-role-httpd)|
|[buluma.mysql](https://galaxy.ansible.com/buluma/mysql)|[![Build Status GitHub](https://github.com/buluma/ansible-role-mysql/workflows/Ansible%20Molecule/badge.svg)](https://github.com/buluma/ansible-role-mysql/actions)|[![Build Status GitLab ](https://gitlab.com/buluma/ansible-role-mysql/badges/main/pipeline.svg)](https://gitlab.com/buluma/ansible-role-mysql)|
|[buluma.openssl](https://galaxy.ansible.com/buluma/openssl)|[![Build Status GitHub](https://github.com/buluma/ansible-role-openssl/workflows/Ansible%20Molecule/badge.svg)](https://github.com/buluma/ansible-role-openssl/actions)|[![Build Status GitLab ](https://gitlab.com/buluma/ansible-role-openssl/badges/main/pipeline.svg)](https://gitlab.com/buluma/ansible-role-openssl)|
|[buluma.php](https://galaxy.ansible.com/buluma/php)|[![Build Status GitHub](https://github.com/buluma/ansible-role-php/workflows/Ansible%20Molecule/badge.svg)](https://github.com/buluma/ansible-role-php/actions)|[![Build Status GitLab ](https://gitlab.com/buluma/ansible-role-php/badges/master/pipeline.svg)](https://gitlab.com/buluma/ansible-role-php)|
|[buluma.python_pip](https://galaxy.ansible.com/buluma/python_pip)|[![Build Status GitHub](https://github.com/buluma/ansible-role-python_pip/workflows/Ansible%20Molecule/badge.svg)](https://github.com/buluma/ansible-role-python_pip/actions)|[![Build Status GitLab ](https://gitlab.com/buluma/ansible-role-python_pip/badges/main/pipeline.svg)](https://gitlab.com/buluma/ansible-role-python_pip)|
|[buluma.selinux](https://galaxy.ansible.com/buluma/selinux)|[![Build Status GitHub](https://github.com/buluma/ansible-role-selinux/workflows/Ansible%20Molecule/badge.svg)](https://github.com/buluma/ansible-role-selinux/actions)|[![Build Status GitLab ](https://gitlab.com/buluma/ansible-role-selinux/badges/main/pipeline.svg)](https://gitlab.com/buluma/ansible-role-selinux)|

## [Context](#context)

This role is a part of many compatible roles. Have a look at [the documentation of these roles](https://buluma.github.io/) for further information.

Here is an overview of related roles:

![dependencies](https://raw.githubusercontent.com/buluma/ansible-role-moodle/png/requirements.png "Dependencies")

## [Compatibility](#compatibility)

This role has been tested on these [container images](https://hub.docker.com/u/buluma):

|container|tags|
|---------|----|
|el|8|
|debian|all|
|fedora|all|
|opensuse|all|
|ubuntu|all|

The minimum version of Ansible required is 2.10, tests have been done to:

- The previous version.
- The current version.
- The development version.



If you find issues, please register them in [GitHub](https://github.com/buluma/ansible-role-moodle/issues)

## [Changelog](#changelog)

[Role History](https://github.com/buluma/ansible-role-moodle/blob/master/CHANGELOG.md)

## [License](#license)

Apache-2.0

## [Author Information](#author-information)

[Michael Buluma](https://buluma.github.io/)
