### PROVISIONING

Dalam hal ini saya melakukan persiapan server (IaC) menggunakan ansible.Saya menggunakan AWS sebagai app server dan Biznet sebagai gateway

berikut adalah scriptnya 


install docker

- name: Install Docker on Ubuntu
  hosts: 13.54.89.240
  become: true
  tasks:

    - name: Update and install required packages
      apt: 
        name: "{{ packages }}"
        state: present
        update_cache: true
      vars:
        packages:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release

    - name: Add Docker's official GPG key
      apt_key:
         url: https://download.docker.com/linux/ubuntu/gpg
         state: present

    - name: Add Docker repository
      apt_repository:
        repo: 
          "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_lsb.codename }} stable"
        update_cache: yes
        state: present

    - name: Install Docker and related packages
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present

    - name: Enable and start Docker services
      ansible.builtin.systemd:
        name: "{{ item }}"
        enabled: true
        state: started
      loop:
        - docker.service
        - containerd.service

    - name: Add user to Docker group
      ansible.builtin.user:
        name: ubuntu
        groups: docker
        append: yes


generate certbot

- name: Letsencrypt Certbot
  hosts: 103.175.221.6
  become: true

  tasks:
    - name: Install Certbot using snap 
      become_user: root 
      shell: |
       sudo snap install --classic certbot &&
       sudo ln -s /snap/bin/certbot /usr/bin/certbot &&
       sudo snap set certbot trust-plugin-with-root=ok &&
       sudo snap install certbot-dns-cloudflare
      register: certbot_install_result
      changed_when: "certbot_install_result.rc == 0"

    - name: Create directory for certs
      file:
        path: "/home/paul/.secrets"
        state: directory

    - name: Create file cloud init in directory secrets 
      template:
        src: ./credentials/cloudflare/cloud_ini.j2
        dest: "/home/paul/.secrets/cloudflare.ini"
        mode: 0400

    - name: Run certbot to generate certificates 
      command: >
        certbot certonly 
        --dns-cloudflare 
        --dns-cloudflare-credentials /home/paul/.secrets/cloudflare.ini 
        --agree-tos
        --non-interactive
        --email sinambelaian@gmail.com
        --domains *.sinambela.studentdumbways.my.id -v
      become_user: root
      timeout: 60
