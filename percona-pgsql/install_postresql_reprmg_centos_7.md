Hướng dẫn cài đặt postgresql Auto Failover Repmgr 
Mô hình
10.0.0.11 Node-1 master (Centos 7)
10.0.0.12 Node-2 slave-1 (Centos 7)
10.0.0.13 Node-3 slave-2 (Centos 7)
10.0.0.14 Haproxy (Ubuntu-20.04)
1. Chuẩn bị môi trường
- Trên tất cả các node cài đặt postgresql
# Sửa file /etc/hosts
nano /etc/hosts
10.0.0.11 node-1
10.0.0.12 node-2
10.0.0.13 node-3
# Cài đặt postgresql trên tất cả các node
    yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm -y
    yum install postgresql96 postgresql96-server postgresql96-contrib postgresql96-libs repmgr96 xinetd -y
    ln -s /usr/pgsql-9.6/bin/repmgr /usr/bin/
# Tao file .bashrc trên tất cả các node
    su postgres && cd 
    touch .bashrc 
    cat <<EOF >> .bashrc 
    # .bashrc
    # User specific aliases and functions
    alias rm='rm -i'
    alias cp='cp -i'
    alias mv='mv -i'
    # Source global definitions
    if [ -f /etc/bashrc ]; then
            . /etc/bashrc
    fi
    source ~/.bash_profile
    EOF
# Tao file .bash_profile trên tất cả các node
    su postgres && cd 
    touch .bash_profile 
    cat <<EOF >> .bash_profile
    [ -f /etc/profile ] && source /etc/profile
    PGDATA=/var/lib/pgsql/9.6/data
    export PGDATA
    export PATH="$PATH:/usr/pgsql-9.6/bin"
    export PS1="[\u@\h \W]\\$ "
    # If you want to customize your settings,
    # Use the file below. This is not overridden
    # by the RPMS.
    [ -f /var/lib/pgsql/.pgsql_profile ] && source /var/lib/pgsql/.pgsql_profile
    alias ssh='ssh -o StrictHostKeyChecking=no'
    alias scp='scp -o StrictHostKeyChecking=no'
    alias rsync='rsync -e "ssh -o StrictHostKeyChecking=no"'
    EOF
# Tao thu muc luu log cho postgres
    mkdir /var/log/postgresql
    chown -R postgres:postgres /var/log/postgresql
# Trên node master
    /usr/pgsql-9.6/bin/postgresql96-setup initdb
# Config postgresql trên node master
sửa các tham số sau trên file: /var/lib/pgsql/9.6/data/postgresql.conf  

        listen_addresses = '*'  
        shared_preload_libraries = 'repmgr'   
        wal_level = replica  
        wal_log_hints = on  
        archive_mode = on
        archive_command = '/var/lib/pgsql/9.6/archives'  
        max_wal_senders = 3  
        max_replication_slots = 2  
        hot_standby = on   
        log_checkpoints = on

# Cấu hình file /etc/repmgr/9.6/repmgr.conf
    cp /etc/repmgr/9.6/repmgr.conf /etc/repmgr/9.6/repmgr.conf_org
    truncate -s 0 /etc/repmgr/9.6/repmgr.conf
    cat <<EOF>> /etc/repmgr/9.6/repmgr.conf
    cluster='pg_cluster'
    node_id=1
    node_name=node-1
    conninfo='host=10.0.0.11  user=repmgr dbname=repmgr connect_timeout=2'
    data_directory='/var/lib/pgsql/9.6/data'
    failover=automatic
    reconnect_attempts=3
    reconnect_interval=5
    promote_command='/usr/pgsql-9.6/bin/repmgr standby promote --log-to-file'
    follow_command='/usr/pgsql-9.6/bin/repmgr standby follow --log-to-file --upstream-node-id=%n'
    log_file='/home/postgres/repmgr/repmgrd-9.6.log'
    event_notifications='standby_promote'
    pg_bindir='/usr/pgsql-9.6/bin/'
    EOF
# Cấu hình file /var/lib/pgsql/9.6/data/pg_hba.conf
Thêm các dòng sau vào cuối file:  

    cat <<EOF>> /var/lib/pgsql/9.6/data/pg_hba.conf
    local   replication     repmgr                                  trust  
    host    replication     repmgr          127.0.0.1/32            trust  
    host    replication     repmgr          10.0.0.11/32            trust  
    host    replication     repmgr          10.0.0.12/32            trust  
    host    replication     repmgr          10.0.0.13/32            trust  
    host    replication     repmgr          10.0.0.14/32            trust  
    local   repmgr          repmgr                                  trust  
    host    repmgr          repmgr          127.0.0.1/32            trust  
    host    repmgr          repmgr          10.0.0.11/32            trust  
    host    repmgr          repmgr          10.0.0.12/32            trust  
    host    repmgr          repmgr          10.0.0.13/32            trust  
    host    repmgr          repmgr          10.0.0.14/32            trust
    EOF  

# Start service postgresql
    systemctl start postgresql-9.6
    systemctl enable postgresql-9.6
# Primary: Register the primary server
    su postgres
    createuser -s repmgr
    createdb repmgr -O repmgr
    repmgr primary register 
    repmgr cluster show
<img src="./images/Screenshot 2021-11-05 235054.png" />  


# Slave-1
# Cấu hình file /etc/repmgr/9.6/repmgr.conf

    cp /etc/repmgr/9.6/repmgr.conf /etc/repmgr/9.6/repmgr.conf_org
    truncate -s 0 /etc/repmgr/9.6/repmgr.conf
    cat <<EOF>> /etc/repmgr/9.6/repmgr.conf
    cluster='pg_cluster'
    node_id=2
    node_name=node-2
    conninfo='host=10.0.0.12  user=repmgr dbname=repmgr connect_timeout=2'
    data_directory='/var/lib/pgsql/9.6/data'
    failover=automatic
    reconnect_attempts=3
    reconnect_interval=5
    promote_command='/usr/pgsql-9.6/bin/repmgr standby promote --log-to-file'
    follow_command='/usr/pgsql-9.6/bin/repmgr standby follow --log-to-file --upstream-node-id=%n'
    log_file='/home/postgres/repmgr/repmgrd-9.6.log'
    event_notifications='standby_promote'
    pg_bindir='/usr/pgsql-9.6/bin/'
    EOF
# Clone  and register standby database tu node master

    repmgr -h 10.0.0.11  -U repmgr -d repmgr standby clone
    systemctl restart postgresql-9.6 &&  systemctl enable postgresql-9.6 # Run on root user
    repmgr standby register
    repmgr cluster show
<img src="./images/Screenshot 2021-11-05 235637.png" />  

# Slave-2
# Cấu hình file /etc/repmgr/9.6/repmgr.conf

    cp /etc/repmgr/9.6/repmgr.conf /etc/repmgr/9.6/repmgr.conf_org
    truncate -s 0 /etc/repmgr/9.6/repmgr.conf
    cat <<EOF>> /etc/repmgr/9.6/repmgr.conf
    cluster='pg_cluster'
    node_id=3
    node_name=node-3
    conninfo='host=10.0.0.13  user=repmgr dbname=repmgr connect_timeout=2'
    data_directory='/var/lib/pgsql/9.6/data'
    failover=automatic
    reconnect_attempts=3
    reconnect_interval=5
    promote_command='/usr/pgsql-9.6/bin/repmgr standby promote --log-to-file'
    follow_command='/usr/pgsql-9.6/bin/repmgr standby follow --log-to-file --upstream-node-id=%n'
    log_file='/home/postgres/repmgr/repmgrd-9.6.log'
    event_notifications='standby_promote'
    pg_bindir='/usr/pgsql-9.6/bin/'
    EOF
# Clone  and register standby database tu node master

    repmgr -h 10.0.0.11  -U repmgr -d repmgr standby clone
    systemctl restart postgresql-9.6 && systemctl enable postgresql-9.6 # Run on root user
    repmgr standby register
    repmgr cluster show
<img src="./images/Screenshot 2021-11-05 235927.png" />  

# High Availability
# Kịch bản node master down --> switch node-2 lên thành master. Node-3 replication từ new master node-2. Rejoin node-1 to cluster
# Bước 1 shutdown node-1 check cluster trên node-2 và node-3
<img src="./images/Screenshot 2021-11-06 000330.png" />  

<img src="./images/Screenshot 2021-11-06 000430.png" />  

# Bước 2: promote node-2 lên thành node master
    repmgr standby promote
<img src="./images/Screenshot 2021-11-06 001040.png" />  

# Bước 3 : Đúng trên node-2 và node-3 check status cluster

<img src="./images/Screenshot 2021-11-06 001120.png" />  

<img src="./images/Screenshot 2021-11-06 001146.png" />  

# Bước 4: Set node-3 follow standby node-2
    repmgr standby follow

<img src="./images/Screenshot 2021-11-06 001355.png" /> 

# Check lại trạng thái cluster
    repmgr cluster show
<img src="./images/Screenshot 2021-11-06 001512.png" /> 

# Bước 4: Start lại server node-1 và join lại vào cluster
# Đứng trên node-3 check lại trạng thái cluster, node-1 đã online tuy nhiên đã bị khai trừ ra khỏi cluster
    repmgr cluster show
<img src="./images/Screenshot 2021-11-06 004803.png" /> 

# Stop service postgresql trên node-1
    systemctl stop postgresql-9.6

# Rejoin cluster
     repmgr node rejoin -d 'host=node-2 dbname=repmgr user=repmgr' --force-rewind --config-files=postgresql.local.conf,postgresql.conf --verbose
     repmgr standby follow
<img src="./images/Screenshot 2021-11-06 005951.png" /> 
<img src="./images/Screenshot 2021-11-06 010837.png" />

# Cài đặt auto switch master
