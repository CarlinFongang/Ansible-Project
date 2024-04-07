# Ansible | Création de roles Ansible et mise en place des roles distant via Gitlab.

_______


><img src="https://i.pinimg.com/280x280_RS/6b/68/be/6b68bed191fdd2fad36e4193e64764ee.jpg" width="50" height="50" alt="Carlin Fongang"> 

> **Carlin FONGANG**  | fongangcarlin@gmail.com

>[LinkedIn](https://www.linkedin.com/in/carlinfongang/) | [GitLab](https://gitlab.com/carlinfongang) | [GitHub](https://github.com/carlinfongang) | [Credly](https://www.credly.com/users/carlin-fongang/badges)

_______

## Contexte
Notre plan d'action a pour but d'améliorer la collaboration avec une autre équipe en transformant notre playbook en un ensemble de rôles modulaires pour une adaptation aisée. Nous intégrerons également un playbook de tests pour faciliter son évaluation et son intégration rapide dans les processus de déploiement existants. De plus, nous créerons un dépôt privé pour nos rôles, permettant ainsi une gestion centralisée pour toute utilisation future. Ces mesures renforceront notre workflow et augmenteront notre capacité à répondre aux besoins spécifiques de l'équipe.

## Objectifs

Dans ce lab, nous allons :

- Rediger des rôles permettant de deployer des conteneurs docker. Cette étape permettra une modularité accrue, offrant ainsi la flexibilité nécessaire pour une adaptation contextuelle.

- Intégrer deux playbook de tests (wordpress et httpd) au rôle pour faciliter son évaluation rapide et son intégration dans le processus de déploiement existant.

- Provisionner une instance cible ec2 à l'aide de terraform et configurer cette instance à l'aide d'ansible et y deployer le nouveau playbook en guise de test.

- Établir une galaxy privée destinée à la conservation et à la gestion centralisée des rôles développés, renforçant l'efficacité et la réutilisation des ressources au sein de notre entreprise.

Ces actions permettront d'optimiser notre workflow et de répondre aux exigences spécifiques de l'équipe.

[Lien vers le playbook à transformer en rôle](https://gitlab.com/CarlinFongang-Labs/Ansible/lab8-role-wordpress.git)
## Prérequis
Disposer d'un machine avec avec ansible déjà installées.

Dans notre cas, nous allons provisionner une instances EC2 s'exécutant sous Ubuntu grace au provider AWS, à partir delaquelle nous effectuerons toutes nos opérations.

Documentation complémentaire :

[Documentation Ansible](https://docs.ansible.com/ansible/latest/index.html)

[Provisionner une instance EC2 sur AWS à l'aide de Terraform](https://gitlab.com/CarlinFongang-Labs/Terraform/lab2-terraform-aws)

[Provisionner une instance EC2 sur AWS à l'aide d'Ansible](https://gitlab.com/CarlinFongang-Labs/Ansible/lab10-deploy-ec2)


## 1. Refactorisation de notre playbook initial en rôle

Rappellons que nous avons mis en place un playbook permettant de lancer un conteneur [Apache consultable ici](https://gitlab.com/CarlinFongang-Labs/Ansible/lab5-playbook.git), et un autre permettant de deployer des conteneur pour [Wordpress conultable ici](https://gitlab.com/CarlinFongang-Labs/Ansible/lab8-role-wordpress.git).

Dans la suite, nous allons transformer les différentes taches du playbook necessaire au deploiement d'un conteneur (par exemple Apache/httpd), en rôle, ainsi nous dechangerons le fichier de playbook de la description de ces taches et nous ferrons simplement appel aux roles, dont nous pourrons modifier le comportement via des variables. 

## 2. Définition des taches

Pour déployer un conteneur, il faut suivre des étapes préparatoires essentielles : 

1. Préparer l'hôte Ansible en installant les paquets nécessaires.

2. Installer Terraform sur la machine Ansible pour gérer le provisionnement de l'infrastructure AWS.

3. Utiliser Terraform pour provisionner l'infrastructure et récupérer l'IP de l'instance créée.

4. Vérifier la connectivité SSH avec la nouvelle instance avant toute opération.

5. Installer les paquets requis, dont Docker, sur l'instance cible.

6. Copier le template Jinja depuis Ansible vers la machine distante.

7. Déployer le conteneur en utilisant le template Jinja et monter les volumes appropriés.

## 3. Création des différents roles

Le playbook initial dont on va s'inspirer pour réaliser nos roles se trouve [ici : Déployer un conteneur Apache (httpd) à l'aide d'un palybook Ansible](https://gitlab.com/CarlinFongang-Labs/Ansible/lab5-playbook.git)

Afin de respecter les normes de création des rôles sous Ansible, nous allons organiser notre projet (**local-role**) en sous repertoire avec une nommenclature spécique comme décrit ci-dessous :

>![Alt text](img/image.png)

Pour donner un peu de contexte : 
Dans un rôle Ansible, chaque dossier et fichier a un but spécifique pour organiser et exécuter les tâches de configuration :

- **tasks/main.yml** : Contient les principales tâches à exécuter par le rôle.
- **handlers/main.yml** : Définit les handlers, des tâches spéciales exécutées en réponse à d'autres tâches.
- **defaults/main.yml** : Spécifie les valeurs par défaut des variables utilisées dans le rôle.
- **vars/main.yml** : Stocke les variables du rôle qui ne devraient pas être modifiées par l'utilisateur.
- **files/** : Contient les fichiers statiques à copier sur les hôtes cibles.
- **templates/** : Stocke les templates Jinja2 qui seront traités et copiés sur les hôtes.
- **meta/main.yml** : Contient les métadonnées du rôle, comme les dépendances.
- **tests/inventory** et **tests/test.yml** : Utilisés pour tester le rôle.

Cette structure aide à la modularité et à la réutilisation des manifest Ansible dans un contexte de travail collaboratif.

Dans notre cas nous mettrons en place et exploiterons une partie de ces repertoires.

### 1. Role "setup_environment"

#### 1.1. Définition du **tasks/main.yml**

Ce rôle Ansible est conçu pour mettre à jour le cache apt et installer une série de paquets essentiels sur la distribution Debian, y compris Ubuntu. Il inclut des outils et librairies couramment utilisés tels qu'unzip, Python, curl, et git, en s'assurant que leur état soit présent sur la machine cible. La tâche devient effective uniquement si le système opérationnel est identifié comme une distribution **Debian**, garantissant ainsi une exécution conditionnelle basée sur l'environnement du système.

Ce scipt sera décrit dans le fichier **local-role/roles/setup_environment/tasks/main.yml**

````bash
---
- name: Update apt cache and install packages
  ansible.builtin.apt:
    name:
      - unzip
      - python3
      - python3-pip
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg-agent
      - software-properties-common
      - wget
      - git
      - gnupg-agent
      - software-properties-common
      - wget
    state: present
    update_cache: true
  become: true
  when: ansible_os_family == "Debian"
````

#### 1.2. Définition du **meta/main.yml**

Le fichier `meta/main.yml` sous Ansible sert à définir les métadonnées d'un rôle. Ces métadonnées incluent des informations telles que les dépendances du rôle (autres rôles nécessaires pour son fonctionnement), la plateforme (systèmes d'exploitation) supportée, et d'autres détails qui aident Ansible Galaxy (le répertoire de rôles Ansible) à indexer et classer le rôle. Ce fichier permet également de spécifier les versions minimales d'Ansible requises pour le rôle, rendant la gestion des dépendances plus transparente et efficace.

Dans notre cas, nous allons hébergé nos rôle sur Gitlab afin de s'en servier comme une sorte de "Galaxy", il est donc necessaire de décrire les fichiers meta pour nos rôles.

Le scipt ci-dessous sera décrit dans le fichier **local-role/roles/setup_environment/meta/main.yml**

````bash
galaxy_info:
  author: Carlin_FONGANG
  description: Update apt cache and install packages on local compute 
  company: Carlin_FONGANG

  license: MIT
  min_ansible_version: 2.16.5

  platforms:
    - name: Ubuntu
      versions:
      - focal    # Ubuntu 20.04
      - bionic   # Ubuntu 18.04
      - jammy    # Ubuntu 22.04
    - name: Debian
      versions:
      - bullseye # Debian 11
      - buster   # Debian 10
  galaxy_tags:
    - terraform
    - aws
dependencies: []
````

#### 1.3.  Définition du **tests/test.yml**

Nous pouvons mettre en palce un playbook de test pour vérfie la fiabilité de ce role 

````bash
nano local-role/roles/setup_environment/tests/test.yml
````

````bash
- hosts: localhost
  become: true
  roles:
    - setup_environment

  tasks:
    - name: Vérifier que les paquets nécessaires sont installés
      ansible.builtin.package_facts:
        manager: auto

    - name: Tester la présence des paquets
      assert:
        that: 
          - "'unzip' in ansible_facts.packages"
          - "'python3' in ansible_facts.packages"
          - "'python3-pip' in ansible_facts.packages"
          - "'apt-transport-https' in ansible_facts.packages"
          - "'ca-certificates' in ansible_facts.packages"
          - "'curl' in ansible_facts.packages"
          - "'gnupg-agent' in ansible_facts.packages"
          - "'software-properties-common' in ansible_facts.packages"
          - "'wget' in ansible_facts.packages"
          - "'git' in ansible_facts.packages"
        fail_msg: "Un ou plusieurs paquets requis ne sont pas installés."
        success_msg: "Tous les paquets requis sont installés."
````

#### 1.4. Test du playbook **setup_environment**

````bash
cd local-role
````

1. Test du role **setup_environment**

````bash
ansible-playbook roles/setup_environment/tests/test.yml -K -vvv
````

Cette commande exécute le playbook Ansible (`test.yml`) situé dans le répertoire `roles/setup_environment/tests`. 

L'option `-K` oblige Ansible d'afficher un prompt,  permettant d'entrer le mot de passe de privilège (sudo) pour exécuter certaines tâches qui nécessitent des privilèges élevés. 

L'option `-vvv` active le mode de verbosité très détaillé, fournissant des informations approfondies sur l'exécution du playbook, utile pour le débogage ou pour comprendre en détail ce qui se passe pendant l'exécution.

2. Résultat du **test**

>![Alt text](img/image-1.png)
*Paquets requis installés*

L'on peut voir que le playbook de test à pu être déployé entièrement et que tous les paquets sont bien installés.


### 2. Role "install_terraform"

Ce rôle Ansible permet de télécharger, installer et vérifier la version de Terraform sur une machine ansible. Il commence par télécharger l'archive de Terraform depuis le site officiel de HashiCorp, en spécifiant la version et l'architecture désirées. Ensuite, il extrait l'archive dans `/usr/local/bin` pour installer Terraform. Enfin, il exécute Terraform pour vérifier l'installation et affiche la version installée, assurant ainsi que Terraform est prêt à être utilisé.

#### 2.1. Définition du **tasks/main.yml**

````bash
nano local-role/roles/install_terraform/tasks/main.yml
````

````bash
---
- name: Download Terraform
  ansible.builtin.get_url:
    url: "https://releases.hashicorp.com/terraform/{{ terraform_version }}/terraform_{{ terraform_version }}_{{ terraform_architecture }}.zip"
    dest: "/tmp/terraform_{{ terraform_version }}_{{ terraform_architecture }}"
  become: true

- name: Unzip Terraform
  ansible.builtin.unarchive:
    src: "/tmp/terraform_{{ terraform_version }}_{{ terraform_architecture }}"
    dest: "/usr/local/bin"
    remote_src: yes
    creates: "/usr/local/bin/terraform"
  become: true

- name: Verify Terraform installation
  ansible.builtin.command: terraform version
  register: terraform_version
  changed_when: false

- name: Show Terraform version
  debug:
      msg: "Terraform version installed: {{ terraform_version.stdout }}"
````

#### 2.2.  Définition du **defaults/main.yml**

````bash
terraform_version: "1.7.5"
terraform_architecture: "linux_amd64"
````

Ces lignes définissent deux variables pour une utilisation avec Terraform dans le fichier de configuration **tasks/main.yml**. 

`terraform_version: "1.7.5"` spécifie la version de Terraform à utiliser, tandis que `terraform_architecture: "linux_amd64"` indique l'architecture cible pour laquelle Terraform est destiné, dans ce cas, Linux 64 bits. Ces variables sont ensuite utilisées pour télécharger la version correcte de Terraform correspondant à ces spécifications.

#### 2.3. Définition du **meta/main.yml**

Pour ce fichier, nous allons simplement reprendre le fichier décrit dans la section  " [1.2. Définition du **meta/main.yml**](https://gitlab.com/CarlinFongang-Labs/Ansible/ansible-project/local-role#12-d%C3%A9finition-du-metamainyml)"

#### 2.4.  Définition du **tests/test_tf.yml**

Ce playbook de test permet de valider l'installation de Terraform sur l'instance Ansible.

````bash
nano roles/install_terraform/tests/test_tf.yml
````

````bash
---
- hosts: localhost
  gather_facts: false
  roles:
    - ../../install_terraform/
  tasks:
    - name: Obtenir la version de Terraform
      command: terraform -v
      register: terraform_version_output

    - name: Afficher la version de Terraform
      debug:
        msg: "La version de Terraform installée est : {{ terraform_version_output.stdout_lines[0] }}"
````


#### 2.5. Test du playbook **install_terraform**

````bash
cd local-role
````

1. Test du role **install_terraform**

````bash
ansible-playbook roles/install_terraform/tests/test_tf.yml -K -vvv
````

2. Résultat du **test_tf**

>![Alt text](img/image-2.png)
*Vérification de la version de terraform*

L'on peut voir que le playbook de test à pu être déployé entièrement et que la version de Terraform souhaitez à bien été installée.


### 3. Role "provide_compute"

#### 3.1. Définition du **tasks/main.yml**
Ce rôle Ansible effectue plusieurs tâches liées à la gestion d'une infrastructure Terraform, notamment le clonage d'un dépôt Git spécifié, l'initialisation et l'application de configurations Terraform dans un répertoire cible, et la gestion d'informations sur une instance EC2. Il commence par cloner un dépôt Git, puis utilise Terraform pour initialiser et appliquer la configuration spécifiée. Enfin, il lit l'adresse IP d'une instance EC2 à partir d'un fichier généré et ajoute cette information à un fichier 'hosts.yml', permettant ainsi une intégration facile avec d'autres opérations Ansible.

````bash
---
- name: Clone repository
  ansible.builtin.git:
    repo: "{{ repo_url }}"
    dest: "{{ repo_dest }}"
    version: "{{ repo_tag }}"
    update: yes

- name: Initialize Terraform
  ansible.builtin.command:
    cmd: terraform init -reconfigure
    chdir: "{{ tf_init_apply_path }}"

- name: Apply Terraform configuration
  ansible.builtin.command:
    cmd: terraform apply -auto-approve
    chdir: "{{ tf_init_apply_path }}"

- name: Read EC2 instance IP from generated file
  ansible.builtin.include_vars:
    file: "{{ ec2_info_file_path }}"
    name: ec2_info

- name: Add EC2 instance IP to 'hosts.yml'
  ansible.builtin.lineinfile:
    path: "{{ hosts_file_path }}"
    line: "          ansible_host: {{ ec2_info.ec2_info.server_public_ip }}"
    create: yes
  delegate_to: localhost

- name: Debug ec2_info
  ansible.builtin.debug:
    var: ec2_info #
````


#### 3.2. Définition du **defaults/main.yml**

````bash
repo_url: "https://gitlab.com/CarlinFongang-Labs/Ansible/tf-provide-target-instance.git"
repo_dest: "/tmp/tf-provide-target-instance"
repo_tag: "main"
tf_init_apply_path: "/tmp/tf-provide-target-instance/ec2_target_host"
ec2_info_file_path: "/tmp/tf-provide-target-instance/ec2_target_host/public_ip.yml"
hosts_file_path: "~/local-role/hosts.yml"
````

Ces variables sont configurées pour le rôle Ansible détaillé plus haut :

- `repo_url` : URL du dépôt Git contenant la configuration Terraform.
- `repo_dest` : Emplacement local où le dépôt sera cloné.
- `repo_tag` : Branche ou tag du dépôt à cloner.
- `tf_init_apply_path` : Dossier contenant les fichiers de configuration Terraform à initialiser et appliquer.
- `ec2_info_file_path` : Chemin vers le fichier stockant l'adresse IP publique de l'instance EC2 générée.
- `hosts_file_path` : Emplacement du fichier `hosts.yml` à mettre qui sera mis à jour avec l'adresse IP de l'instance EC2 une fois celle ci provisionné à l'aide de Terraform.

toutes ces variables sont ajustable dans le fichier **group_vars/all.yml**, selon le contexte d'utilisation

#### 3.2. Définition du **meta/main.yml**

Pour ce fichier, nous allons simplement reprendre le fichier décrit dans la section  " [1.2. Définition du **meta/main.yml**](https://gitlab.com/CarlinFongang-Labs/Ansible/ansible-project/local-role#12-d%C3%A9finition-du-metamainyml)"

#### 3.3.  Définition du **tests/test_provide.yml**

Ce playbook de test permet de valider le bon fonctionneement du role **provide_compute** .

````bash
nano roles/provide_compute/tests/test_provide.yml
````

````bash
---
- hosts: localhost
  gather_facts: false
  roles:
    - ../../provide_compute

  tasks:
    - name: Afficher l'adresse IP de l'instance EC2
      debug:
        msg: "L'adresse IP de l'instance EC2 est : {{ ec2_info.ec2_info.server_public_ip }}"
````


#### 3.4. Test du playbook **provide_compute**

Afin de tester ce playbook, nous aurons besoin de mettre en place un bucker "s3" pour le stockage du Remote state, et répérer également la paire de clé de sécurité de l'utilisateur AWS.

une fois ces informations récupérées : 

1. Exporter la paire de clé

````bash
export AWS_ACCESS_KEY_ID="AKIAQ2LRBUZA7D32KAX3"
export AWS_SECRET_ACCESS_KEY="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"
````

remplacer les valeurs par ceux du compte AWS à utiliser

2. Créer un bucket **s3**

L'usage du manifest Terraform défini par défaut dans le role, est conditionné par l'existance d'un bucket où sera stocké le **tfstate**, le nom de ce bucket est consultable dans le code Terraform [Terraform - Provisionnement instance cible](https://gitlab.com/CarlinFongang-Labs/Ansible/tf-provide-target-instance/-/blob/main/ec2_target_host/main.tf)

>![Alt text](img/image-3.png)

Noté qu'il est également possible de **forker** le projet Terraform et de modifier le nom du bucket pour l'ajuster à un contexte spécifique, il faudra alors modifier la variable **repo_url** dans le fichier **group_vars/all.yml**, afin de définir le nouveau dépot de code terraform

Une fois ces conditions préalables remplies, on peut tester le playbook 

3. Test du role **provide_compute**
````bash
ansible-playbook roles/provide_compute/tests/test_provide.yml -K -vvv
````
4. Résultat du **test_provide**

>![Alt text](img/image-4.png)
*Instance provisionnée et ip disponible*

>![Alt text](img/image-5.png)
*Instance provisionnée*

L'on peut voir que le playbook de test à pu être déployé entièrement et qu'une instance ec2 à été provisionnée ainsi qu'une adresse Elastic IP.

De plus ce role permet d'écrire directement l'EIP de la nouvelle instance dans le fichier hosts.yml, ce qui va être pratique par la suite pour effectuer les diverses opérations sur notre nouvelle machine distante.


### 4. Role "check_ssh"

#### 4.1. Définit du **tasks/main.yml**

Le rôle `check_ssh` est conçu pour vérifier la disponibilité du service SSH sur des serveurs. Il utilise la tâche `wait_for` pour attendre que le port 22 (SSH) sur les hôtes spécifiés soit accessible, assurant ainsi que les serveurs sont prêts pour des connexions SSH. Cette vérification est effectuée sur tous les hôtes listés dans le groupe `dynamic_hosts`. 

````bash
---
- name: Refresh Inventory to Ensure Latest Hosts are Used
  meta: refresh_inventory

- name: Wait for SSH to come up on the servers
  ansible.builtin.wait_for:
    host: "{{ hostvars[item].ansible_host }}"
    port: 22
    state: started
  loop: "{{ groups['dynamic_hosts'] }}"
  when: "'dynamic_hosts' in groups"
````

#### 4.2. Définition du **meta/main.yml**

Pour ce fichier, nous allons simplement reprendre le fichier décrit dans la section  " [1.2. Définition du **meta/main.yml**](https://gitlab.com/CarlinFongang-Labs/Ansible/ansible-project/local-role#12-d%C3%A9finition-du-metamainyml)"

#### 4.3.  Définition du **tests/test_check_ssh.yml**

Nous avons provisionné une instance ec2 dans l'étape précédente, nous allons à présent vérifier qu'il est possible de se connecter en ssh à cette instance 

````bash
nano roles/check_ssh/tests/test_check_ssh.yml
````

````bash
---
- hosts: dynamic_hosts
  gather_facts: false
  roles:
    - ../../check_ssh
  tasks:
    - name: Afficher le message de réussite
      debug:
        msg: "Connexion ssh établie avec succes !"
      when: ansible_failed_task is undefined
````

#### 4.4. Test du playbook **test_check_ssh**

````bash
ansible-playbook roles/check_ssh/tests/test_check_ssh.yml -K -vvv
````

>![Alt text](img/image-6.png)
*Connexion ssh réussie*


### 5. Role "prepare_docker"

#### 5.1. Définit du **tasks/main.yml**

Ce rôle Ansible est conçu pour installer Docker  et ses dépendances sur un système. Il commence par mettre à jour le cache des paquets et installer des prérequis. Ensuite, il ajoute la clé GPG officielle de Docker et installe Docker en utilisant un script récupéré depuis le site officiel de Docker. Le playbook ajoute également l'utilisateur courant au groupe Docker pour permettre l'exécution de commandes Docker sans sudo. Enfin, il assure l'installation de pip et installe Docker Compose, en vérifiant la version installée.


````bash
---
- name: Install Docker and dependencies
  ansible.builtin.apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - software-properties-common
      - gnupg-agent
    state: present
    update_cache: yes

- name: Add Docker's official GPG key
  ansible.builtin.apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present
  when: ansible_os_family == "Debian"

- name: Install Docker
  ansible.builtin.shell: curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh

- name: Add current user to Docker group
  ansible.builtin.user:
    name: "{{ ansible_env.SUDO_USER | default(ansible_env.USER) }}"
    groups: docker
    append: true

- name: Ensure pip is installed
  ansible.builtin.package:
    name: python3-pip
    state: present
  become: true

- name: Install Docker Compose
  block:
    - name: Download Docker Compose binary
      ansible.builtin.get_url:
        url: "https://github.com/docker/compose/releases/download/{{ docker_compose_version }}/docker-compose-{{ ansible_system }}-{{ ansible_architecture }}"
        dest: "/usr/local/bin/docker-compose"
        mode: '0755'
      become: true

    - name: Verify Docker Compose installation
      command: docker-compose --version
      register: docker_compose_version
      failed_when: docker_compose_version.rc != 0

    - name: Output Docker Compose version
      debug:
        msg: "Docker Compose version installed: {{ docker_compose_version.stdout }}"
  when: ansible_os_family == "Debian" or ansible_os_family == "RedHat"

````


#### 5.2. Définition du **defaults/main.yml**
Dans ce fichier, nous avons déclarer la version de docker-compose à installer, la variable `docker_compose_version` est modifiable dans le fichier **group_vars/all.yml**

````bash
docker_compose_version: "1.29.2" 
`````

#### 5.2. Définition du **meta/main.yml**

Pour ce fichier, nous allons simplement reprendre le fichier décrit dans la section  " [1.2. Définition du **meta/main.yml**](https://gitlab.com/CarlinFongang-Labs/Ansible/ansible-project/local-role#12-d%C3%A9finition-du-metamainyml)"

#### 5.3.  Définition du **tests/test_docker_install.yml**

````bash
- name: Deploy Apache container
  hosts: dynamic_hosts
  become: true
  vars_files:
    - group_vars/all.yml
  roles:
    - ../../prepare_docker
````

#### 5.4. Test du playbook **prepare_docker**

````bash
ansible-playbook roles/prepare_docker/tests/test_docker_install.yml -K -vvv
````

>![Alt text](img/image-7.png)
*Bonne exécution du playbook*

>![Alt text](img/image-8.png)
*Vérification de docker sur l'instance cible*

### 6. Role "copy_template"

#### 6.1. Définit du **tasks/main.yml**

Ce rôle utilise le module `ansible.builtin.template` pour copier un modèle de fichier web, spécifiquement `index.html.j2`, vers un chemin défini sur la machine distante. Il définit également les permissions du fichier cible à '0644', assurant ainsi que le fichier est lisible par tous mais modifiable uniquement par le propriétaire. Ce processus permet de personnaliser et de déployer facilement des fichiers de configuration ou des pages web sur des serveurs distants.

````bash
- name: Copy website file template
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ remote_template_path }}"
    mode: '0644'
````

#### 6.2. Définition du **defaults/main.yml**

````bash
remote_template_path: "/home/{{ ansible_env.SUDO_USER | default(ansible_env.USER) }}/index.html" #1
````

#### 6.2. Définition du **meta/main.yml**

Pour ce fichier, nous allons simplement reprendre le fichier décrit dans la section  " [1.2. Définition du **meta/main.yml**](https://gitlab.com/CarlinFongang-Labs/Ansible/ansible-project/local-role#12-d%C3%A9finition-du-metamainyml)"

#### 6.3.  Définition du **tests/test_copy.yml**

````bash
nana roles/copy_template/tests/test_copy.yml
````

````bash
- name: Copy Template jinja on target instance
  hosts: dynamic_hosts
  become: true
  roles:
    - ../../copy_template
  tasks:
    - name: Afficher le message de réussite
      debug:
        msg: "Opération réussie !"
      when: ansible_failed_task is undefined
````

#### 6.4. Test du playbook **test_copy**

````bash
ansible-playbook roles/copy_template/tests/test_copy.yml -K -vvv
````

>![Alt text](img/image-9.png)
*Opération de copie réussie*

En se connectant sur l'instance cible, on peut constater que le fichier est bel et bien copié

>![Alt text](img/image-10.png)
*fichier index.html copié sur l'intance cible*


### 6. Role "deploy_container"

#### 6.1. Définit du **tasks/main.yml**

Ce rôle Ansible utilise le module `community.docker.docker_container` pour télécharger et exécuter un conteneur Docker spécifié. Il définit le nom du conteneur, l'image à utiliser, les ports à exposer, et monte un volume pour le contenu web. Le conteneur est configuré pour démarrer automatiquement et adhérer à une politique de redémarrage continue, garantissant sa disponibilité constante.

````bash
---
- name: Pull and run a Docker container
  community.docker.docker_container:
    name: "{{ container_name }}"
    image: "{{ container_image }}"
    ports: "{{ container_ports }}"
    volumes:
      - "{{ remote_template_path }}:{{ document_root }}/index.html"
    state: started
    restart_policy: always
````

#### 6.2. Définition du **defaults/main.yml**

````bash
container_name: webapp
container_image: httpd
container_ports:
  - "80:80"
document_root: "/usr/local/apache2/htdocs" #2
remote_template_path: "/home/{{ ansible_env.SUDO_USER | default(ansible_env.USER) }}/index.html"
````

Ces variables sont configurées pour le rôle visant à déployer un conteneur Docker :

- `container_name`: Identifie le nom du conteneur, ici `webapp`.
- `container_image`: Spécifie l'image Docker à utiliser, dans ce cas `httpd` pour un serveur web Apache.
- `container_ports`: Définit les ports à exposer, mappant le port 80 du conteneur sur le port 80 de l'hôte.
- `document_root`: Le chemin où les fichiers du site web doivent être stockés dans le conteneur, `/usr/local/apache2/htdocs` étant le répertoire racine d'Apache.

ces variables sont ajustable dans le fichier **group_vars/all.yml** selon le contexte.

#### 6.2. Définition du **meta/main.yml**

Pour ce fichier, nous allons simplement reprendre le fichier décrit dans la section  " [1.2. Définition du **meta/main.yml**](https://gitlab.com/CarlinFongang-Labs/Ansible/ansible-project/local-role#12-d%C3%A9finition-du-metamainyml)"

#### 6.3.  Définition du **tests/test_deploy_container.yml**

````bash
---
- name: Deploy contenair
  hosts: dynamic_hosts
  become: true
  roles:
    - ../../deploy_container
  tasks:
    - name: Afficher le message de réussite
      debug:
        msg: "Opération réussie !"
      when: ansible_failed_task is undefined
````

#### 6.4. Test du playbook **test_deploy_container**

````bash
ansible-playbook roles/deploy_container/tests/test.yml -K -vvv
````

>![Alt text](img/image-11.png)
*Déploiement du conteneur réussi*

>![Alt text](img/image-12.png)

En se connectant à l'intance cible, on peut voir que le conteneur est en cours

1. Vérification via navigateur

>![Alt text](img/image-13.png)
*L'application webapp ainsi que le template jinja son bien déployé*

## 3. Définition de playbook intégrant les roles

### 3.1. Playbook de déploiement d'un conteneur httpd + template jinja : "deploy_httpd.yml"

1. Description du playbook : 

````bash
---
- name: Setup environment
  hosts: localhost
  roles:
    - setup_environment

- name: Install Terraform & provide instance
  hosts: localhost
  roles:
    - install_terraform
    - provide_compute
  tasks:
    - name: Refresh Inventory to Ensure Latest Hosts are Used
      meta: refresh_inventory

- name: Verify SSH Connectivity on Dynamic Hosts
  hosts: dynamic_hosts
  gather_facts: no
  roles:
    - check_ssh

- name: Deploy Apache container
  hosts: dynamic_hosts
  become: true
  roles:
    - prepare_docker
    - copy_template
    - deploy_container
````

Ce playbook Ansible est structuré en plusieurs phases pour configurer un environnement, installer Terraform, provisionner une instance, vérifier la connectivité SSH aux hôtes dynamiques, et déployer un conteneur Apache. Il utilise des rôles spécifiques pour chaque étape, définis et documentés précédement. Il assure aussi la mise à jour de l'inventaire pour inclure les hôtes récemment provisionnés.

2. Déploiement du playbook

````bash
ansible-playbook deploy_httpd.yml -K -vvv
````

>![Alt text](img/image-14.png)
*Le Mot de passe de l'utilisateur sudo est demandé pour l'opération*

>![Alt text](img/image-15.png)
*L'installation est achevé et une nouvelle adresse est renseigné dans le fichier hosts*


3. Résultat du déploiement

>![Alt text](img/image-16.png)
*Conteneur httpd et template jinja déployé*

### 3.2. Playbook de déploiement de conteneurs Wordpress

1. Description du playbook : 

````bash
---
- name: Setup environment
  hosts: localhost
  roles:
    - setup_environment

- name: Install Terraform & provide instance
  hosts: localhost
  roles:
    - install_terraform
    - provide_compute
  tasks:
    - name: Refresh Inventory to Ensure Latest Hosts are Used
      meta: refresh_inventory

- name: Verify SSH Connectivity on Dynamic Hosts
  hosts: dynamic_hosts
  gather_facts: no
  roles:
    - check_ssh

- name: Deploy Wordpress
  hosts: dynamic_hosts
  become: true
  roles:
    - wordpress
````

Ce playbook Ansible est structuré en plusieurs phases pour configurer un environnement, installer Terraform, provisionner une instance, vérifier la connectivité SSH aux hôtes dynamiques, et déployer un conteneur Wordpress. Il utilise des rôles spécifiques pour chaque étape, définis et documentés précédement. Il assure aussi la mise à jour de l'inventaire pour inclure les hôtes récemment provisionnés.

2. Déploiement du playbook

````bash
ansible-galaxy install -r roles/requirements.yml -f
````

````bash
ansible-playbook deploy_wordpress.yml -K -vvv
````

>![Alt text](img/image-17.png)
*Exécution du playbook*

>![Alt text](img/image-18.png)
*Ajout de l'eip de la nouvelle instance au fichier hosts*

3. Résultat

En se connectant à l'instance cible, l'on peux intéroger les conteneur en cours d'exécution

>![Alt text](img/image-20.png)
*Conteneurs Wordpress en corus*

>![Alt text](img/image-19.png)
*Interface Wordpress*

## 3. Définition de playbook intégrant les roles distant (hébergés sur gitlab)

Pour déployer de nouveaux playbooks utilisant des rôles définis précédemment, mais en remote, semblable à "Ansible Galaxy", ces rôles seront hébergés sur GitLab à l'adresse : [RolesHub](https://gitlab.com/CarlinFongang-Labs/Ansible/Roleshub).

Après avoir hébergé les rôles sur GitLab, le projet sera restructuré en actualisant le fichier  **roles/requirement.yml** avec les sources des rôles et en supprimant les sous-répertoires inutiles du dossier **roles**. La nouvelle structure du projet sera ainsi simplifiée.

La nouvelle strucutre du projet sera alors la suivante : 

![Alt text](img/image-26.png)

1. Définintion du fichier **roles/requirement.yml**

````bash
roles:
  - src: https://gitlab.com/CarlinFongang-Labs/Ansible/Roleshub/setup_environment.git
    scm: git
    version: "main"
    name: setup_environment

  - src: https://gitlab.com/CarlinFongang-Labs/Ansible/Roleshub/install_terraform.git
    scm: git
    version: "main"
    name: install_terraform

  - src: https://gitlab.com/CarlinFongang-Labs/Ansible/Roleshub/provide_compute.git
    scm: git
    version: "main"
    name: provide_compute

  - src: https://gitlab.com/CarlinFongang-Labs/Ansible/Roleshub/check_ssh.git
    scm: git
    version: "main"
    name: check_ssh

  - src: https://gitlab.com/CarlinFongang-Labs/Ansible/Roleshub/prepare_docker.git
    scm: git
    version: "main"
    name: prepare_docker

  - src: https://gitlab.com/CarlinFongang-Labs/Ansible/Roleshub/copy_template.git
    scm: git
    version: "main"
    name: copy_template

  - src: https://gitlab.com/CarlinFongang-Labs/Ansible/Roleshub/deploy_container.git
    scm: git
    version: "main"
    name: deploy_container

````

Ce fichier décrit plusieurs rôles Ansible, chacun provenant de dépôts GitLab, avec des informations sur le type de gestion de configuration (Git), la version (tous sur "main"), et leurs noms. Ces rôles couvrent diverses tâches comme la configuration de l'environnement, l'installation de Terraform, la vérification de la connectivité SSH, la préparation de Docker, la copie de modèles de site web, et le déploiement de conteneurs Docker, facilitant la gestion automatisée d'infrastructures et la réutilisabilité des roles.

2. Exécution du playbook

````bash
ansible-galaxy install -r roles/requirements.yml -f
````

>![Alt text](img/image-21.png)

````bash
ansible-playbook deploy_httpd.yml -k -vvv
````

>![Alt text](img/image-22.png)
*Déploiment du conteneur httpd achevé*

>![Alt text](img/image-23.png)
*Le conteneur est en cours d'exécution*

>![Alt text](img/image-24.png)
*L'application est consultable depuis le navigateur*

Dépôt du projet utilisant les roles en remote (roles distant sur Gitlab) :  [Remote Role](https://gitlab.com/CarlinFongang-Labs/Ansible/ansible-project/remote-role.git)

