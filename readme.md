# Práctica 3.1. Implantación de Moodle en Amazon Web Services (AWS) mediante Ansible

Este repositorio contiene varios playbooks de Ansible para configurar los componentes necesarios para instalar y configurar Moodle, MySQL, Apache y Certbot con Let's Encrypt en un servidor backend y frontend.

## Requisitos

- **Ansible**: Para ejecutar los playbooks.
- **Sistema operativo**: Ubuntu 20.04 o posterior.
- **Acceso de superusuario**: Todos los playbooks requieren privilegios elevados para realizar cambios en el sistema.

## Playbooks

### 1. **Configuración de base de datos para Moodle**

Este playbook configura la base de datos necesaria para Moodle.

#### Tareas:

- Elimina la base de datos de Moodle si existe.
- Crea una nueva base de datos para Moodle.
- Elimina el usuario de Moodle si existe.
- Crea un nuevo usuario con privilegios para Moodle.
- Actualiza los privilegios del usuario en la base de datos.

```yaml
- name: Configuración de base de datos para Moodle
  hosts: backend
  become: yes
  vars_files:
    - ../vars/variables.yml
  tasks:
    - name: Eliminar la base de datos si existe
      mysql_db:
        name: "{{ moodle.db.name }}"
        state: absent
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Crear la base de datos
      mysql_db:
        name: "{{ moodle.db.name }}"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Eliminar el usuario si existe
      mysql_user:
        name: "{{ moodle.db.user }}"
        host: "{{ ips.frontend_private }}"
        state: absent
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Crear el usuario para Moodle
      mysql_user:
        name: "{{ moodle.db.user }}"
        host: "{{ ips.frontend_private }}"
        password: "{{ moodle.db.pass }}"
        priv: "{{ moodle.db.name }}.*:ALL"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Actualizar privilegios
      mysql_user:
        name: "{{ moodle.db.user }}"
        host: "{{ ips.frontend_private }}"
        check_implicit_admin: yes
        login_unix_socket: /var/run/mysqld/mysqld.sock
```
## 2. **Configuración de MySQL**

Este playbook instala y configura MySQL en el servidor backend.

#### Tareas:

- Actualiza los repositorios.
- Actualiza los paquetes instalados.
- Instala MySQL Server.
- Instala el módulo Python para MySQL.
- Configura el archivo `mysqld.cnf` para vincular MySQL a la IP del servidor.
- Reinicia el servicio de MySQL.

```yaml
- name: Configuración de MySQL
  hosts: backend
  become: yes
  vars_files:
    - ../vars/variables.yml
  tasks:
    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Actualizar los paquetes instalados
      apt:
        upgrade: dist
        autoremove: yes

    - name: Instalar MySQL Server
      apt:
        name: mysql-server
        state: present

    - name: Instalamos el módulo de pymysql
      apt:
        name: python3-pymysql
        state: present

    - name: Configurar el archivo mysqld.cnf
      lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: "^bind-address\\s*=\\s*.*"
        line: "bind-address = {{ ips.backend_private }}"
        state: present
        
    - name: Reiniciar el servicio de MySQL
      service:
        name: mysql
        state: restarted
```
### 3. **Instalación y configuración de MySQL Server**

Este playbook instala y configura MySQL en el servidor backend.

#### Tareas:

- Actualiza los repositorios.
- Actualiza los paquetes instalados.
- Instala MySQL Server.
- Instala el módulo Python para MySQL.
- Configura el archivo `mysqld.cnf` para vincular MySQL a la IP del servidor.
- Reinicia el servicio de MySQL.

```yaml
- name: Configuración de MySQL
  hosts: backend
  become: yes
  vars_files:
    - ../vars/variables.yml
  tasks:
    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Actualizar los paquetes instalados
      apt:
        upgrade: dist
        autoremove: yes

    - name: Instalar MySQL Server
      apt:
        name: mysql-server
        state: present

    - name: Instalamos el módulo de pymysql
      apt:
        name: python3-pymysql
        state: present

    - name: Configurar el archivo mysqld.cnf
      lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: "^bind-address\\s*=\\s*.*"
        line: "bind-address = {{ ips.backend_private }}"
        state: present
        
    - name: Reiniciar el servicio de MySQL
      service:
        name: mysql
        state: restarted
```
### 4. **Configuración del servidor web Apache y PHP**

Este playbook instala y configura Apache junto con PHP y los módulos necesarios en el servidor frontend.

#### Tareas:

- Actualiza los repositorios.
- Actualiza los paquetes instalados.
- Instala Apache.
- Instala el paquete `unzip`.
- Habilita el módulo `rewrite` de Apache.
- Copia el archivo de configuración de Apache.
- Instala PHP y los módulos necesarios para Moodle.
- Reinicia el servicio de Apache.
- Copia un archivo de prueba PHP.
- Cambia el propietario y grupo del archivo `index.php`.

```yaml
- name: Configurar servidor web Apache y PHP
  hosts: frontend
  become: yes
  tasks:
    - name: Actualizar repositorios
      apt:
        update_cache: yes

    - name: Actualizar paquetes
      apt:
        upgrade: dist

    - name: Instalar Apache
      apt:
        name: apache2
        state: present

    - name: Instalar unzip
      apt:
        name: unzip
        state: present

    - name: Habilitar módulo rewrite de Apache
      command: a2enmod rewrite

    - name: Copiar archivo de configuración de Apache
      copy:
        src: ../templates/000-default.conf
        dest: /etc/apache2/sites-available/000-default.conf

    - name: Instalar PHP y módulos necesarios
      apt:
        name:
          - php
          - php-mysql
          - libapache2-mod-php
          - php-xml
          - php-mbstring
          - php-curl
          - php-zip
          - php-gd
          - php-intl
          - php-soap
        state: present

    - name: Reiniciar servicio de Apache
      service:
        name: apache2
        state: restarted

    - name: Copiar script de prueba de PHP
      copy:
        src: ../php/index.php
        dest: /var/www/html/index.php

    - name: Cambiar propietario y grupo del archivo index.php
      file:
        path: /var/www/html/index.php
        owner: www-data
        group: www-data
        mode: "0644"
```

### 5. **Instalación y configuración de Moodle**

Este playbook instala y configura Moodle en el servidor frontend, configurando Apache y PHP, descargando Moodle, configurando su base de datos y ejecutando la instalación desde la línea de comandos.

#### Tareas:

- Habilita el módulo `rewrite` en Apache.
- Elimina instalaciones previas de Moodle.
- Crea el directorio para Moodle.
- Descarga y descomprime el archivo zip de Moodle.
- Copia los archivos de Moodle al directorio web.
- Cambia los permisos del directorio de Moodle.
- Crea el directorio para los datos de Moodle.
- Modifica la configuración de PHP.
- Instala Moodle desde la CLI.
- Reinicia Apache.

```yaml
- name: Instalar y configurar Moodle
  hosts: frontend
  become: true
  vars_files:
    - ../vars/variables.yml  # Cargar variables desde un archivo externo

  tasks:
    - name: Habilitar módulo rewrite en Apache
      ansible.builtin.command: a2enmod rewrite

    - name: Eliminar instalaciones previas de Moodle
      ansible.builtin.file:
        path: "{{ item }}"
        state: absent
      loop:
        - "{{ routes.moodle_directory }}"
        - "{{ moodle.dataroot }}"
        - "/tmp/{{ moodle.version }}.zip"

    - name: Crear directorio de Moodle
      ansible.builtin.file:
        path: "{{ routes.moodle_directory }}"
        state: directory
        mode: '0755'

    - name: Descargar archivo zip de Moodle
      ansible.builtin.get_url:
        url: https://github.com/moodle/moodle/archive/refs/tags/v4.3.1.zip
        dest: "/tmp/{{ moodle.version }}.zip"

    - name: Descomprimir Moodle
      ansible.builtin.unarchive:
        src: "/tmp/{{ moodle.version }}.zip"
        dest: "/tmp"
        remote_src: yes

    - name: Copiar archivos de Moodle al directorio web
      ansible.builtin.copy:
        src: "/tmp/moodle-{{ moodle.version }}/"
        dest: "{{ routes.moodle_directory }}"
        remote_src: yes

    - name: Cambiar propietario y permisos del directorio de Moodle
      ansible.builtin.file:
        path: "{{ routes.moodle_directory }}"
        owner: "www-data"
        group: "www-data"
        mode: '0755'
        recurse: yes

    - name: Crear directorio para los datos de Moodle
      ansible.builtin.file:
        path: "{{ moodle.dataroot }}"
        state: directory
        owner: "www-data"
        group: "www-data"
        mode: '0777'

    - name: Modificar configuración de PHP
      ansible.builtin.lineinfile:
        path: "{{ item }}"
        regexp: "^;max_input_vars = 1000"
        line: "max_input_vars = 5000"
        state: present
      loop:
        - "{{ routes.apache_ini }}"
        - "{{ routes.cli_ini }}"

    - name: Instalar Moodle desde CLI
      ansible.builtin.shell: |
        php {{ routes.moodle_directory }}/admin/cli/install.php \
          --lang={{ moodle.lang }} \
          --wwwroot={{ moodle.wwwroot }} \
          --dataroot={{ moodle.dataroot }} \
          --dbtype={{ moodle.db.type }} \
          --dbhost={{ moodle.db.host }} \
          --dbname={{ moodle.db.name }} \
          --dbuser={{ moodle.db.user }} \
          --dbpass={{ moodle.db.pass }} \
          --fullname="{{ moodle.fullname }}" \
          --shortname="{{ moodle.shortname }}" \
          --summary="{{ moodle.summary }}" \
          --adminuser={{ moodle.admin.user }} \
          --adminpass={{ moodle.admin.pass }} \
          --adminemail={{ moodle.admin.email }} \
          --non-interactive \
          --agree-license
      become: true

    - name: Cambiar propietario de los archivos de Moodle
      ansible.builtin.file:
        path: "{{ routes.moodle_directory }}"
        owner: "www-data"
        group: "www-data"
        recurse: yes

    - name: Reiniciar Apache
      ansible.builtin.service:
        name: apache2
        state: restarted
