# 一、说明
                
   * 全新安装的 Centos7 系统

   * 保持服务器网络畅通

# 二、安装
  * 以下命令均在一个终端里面执行

        $ systemctl enable firewalld \
          && systemctl start firewalld \
          && firewall-cmd --zone=public --add-port=80/tcp --permanent \
          && firewall-cmd --zone=public --add-port=2222/tcp --permanent \
          && firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="172.17.0.0/16" port protocol="tcp" port="8080" accept" \
          && firewall-cmd --reload \
          && setenforce 0 \
          && sed -i "s/enforcing/disabled/g" /etc/selinux/config

*
        $ yum update -y \
          && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
          && yum -y install kde-l10n-Chinese \
          && yum -y reinstall glibc-common \
          && localedef -c -f UTF-8 -i zh_CN zh_CN.UTF-8 \
          && export LC_ALL=zh_CN.UTF-8 \
          && echo 'LANG="zh_CN.UTF-8"' > /etc/locale.conf \
          && yum -y install wget gcc epel-release git \
          && yum install -y yum-utils device-mapper-persistent-data lvm2 \
          && yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo \
          && yum makecache fast \
          && rpm --import https://mirrors.aliyun.com/docker-ce/linux/centos/gpg \
          && echo -e "[nginx]\nname=nginx repo\nbaseurl=http://nginx.org/packages/centos/7/\$basearch/\ngpgcheck=0\nenabled=1\n" > /etc/yum.repos.d/nginx.repo \
          && yum -y install redis mariadb mariadb-devel mariadb-server nginx docker-ce \
          && systemctl enable redis mariadb nginx docker \
          && systemctl start redis mariadb \
          && yum -y install python36 python36-devel \
          && python3.6 -m venv /opt/py3

*
        $ cd /opt \
          && git clone https://github.com/jumpserver/jumpserver.git \
          && wget https://github.com/jumpserver/luna/releases/download/1.4.6/luna.tar.gz \
          && yum -y install $(cat /opt/jumpserver/requirements/rpm_requirements.txt) \
          && source /opt/py3/bin/activate \
          && pip install --upgrade pip setuptools \
          && pip install -r /opt/jumpserver/requirements/requirements.txt \
          && curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io \
          && systemctl restart docker \
          && docker pull jumpserver/jms_coco:1.4.6 \
          && docker pull jumpserver/jms_guacamole:1.4.6 \
          && cd /opt \
          && tar xf luna.tar.gz \
          && chown -R root:root luna \
          && rm -rf /etc/nginx/conf.d/default.conf

*
        $ cat << EOF > /etc/nginx/conf.d/jumpserver.conf
        server {
            listen 80;

            client_max_body_size 100m;  # 录像及文件上传大小限制

            location /luna/ {
                try_files \$uri / /index.html;
                alias /opt/luna/;
            }

            location /media/ {
                add_header Content-Encoding gzip;
                root /opt/jumpserver/data/;
            }

            location /static/ {
                root /opt/jumpserver/data/;
            }

            location /socket.io/ {
                proxy_pass       http://localhost:5000/socket.io/;
                proxy_buffering off;
                proxy_http_version 1.1;
                proxy_set_header Upgrade \$http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_set_header X-Real-IP \$remote_addr;
                proxy_set_header Host \$host;
                proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
                access_log off;
            }

            location /coco/ {
                proxy_pass       http://localhost:5000/coco/;
                proxy_set_header X-Real-IP \$remote_addr;
                proxy_set_header Host \$host;
                proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
                access_log off;
            }

            location /guacamole/ {
                proxy_pass       http://localhost:8081/;
                proxy_buffering off;
                proxy_http_version 1.1;
                proxy_set_header Upgrade \$http_upgrade;
                proxy_set_header Connection \$http_connection;
                proxy_set_header X-Real-IP \$remote_addr;
                proxy_set_header Host \$host;
                proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
                access_log off;
            }

            location / {
                proxy_pass http://localhost:8080;
                proxy_set_header X-Real-IP \$remote_addr;
                proxy_set_header Host \$host;
                proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            }
        }
        EOF

*
        $ systemctl start nginx \
          && DB_PASSWORD=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 24` \
          && BOOTSTRAP_TOKEN=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 16` \
          && cp /opt/jumpserver/config_example.py /opt/jumpserver/config.py \
          && Server_IP=`ip addr | grep inet | egrep -v '(127.0.0.1|inet6|docker)' | awk '{print $2}' | tr -d "addr:" | head -n 1 | cut -d / -f1` \
          && mysql -uroot -e "create database jumpserver default charset 'utf8';grant all on jumpserver.* to 'jumpserver'@'127.0.0.1' identified by '$DB_PASSWORD';flush privileges;" \
          && sed -i "s/BOOTSTRAP_TOKEN = 'PleaseChangeMe'/BOOTSTRAP_TOKEN = '$BOOTSTRAP_TOKEN'/g" /opt/jumpserver/config.py \
          && sed -i "s/# DEBUG = True/DEBUG = False/g" /opt/jumpserver/config.py \
          && sed -i "s/# LOG_LEVEL = 'DEBUG'/LOG_LEVEL = 'ERROR'/g" /opt/jumpserver/config.py \
          && sed -i "s/# SESSION_EXPIRE_AT_BROWSER_CLOSE = False/SESSION_EXPIRE_AT_BROWSER_CLOSE = True/g" /opt/jumpserver/config.py \
          && sed -i "s/DB_PASSWORD = ''/DB_PASSWORD = '$DB_PASSWORD'/g" /opt/jumpserver/config.py

*
        $ cd /opt/jumpserver \
          && ./jms start all -d \
          && docker run --name jms_coco -d -p 2222:2222 -p 5000:5000 -e CORE_HOST=http://$Server_IP:8080 -e BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN jumpserver/jms_coco:1.4.6 \
          && docker run --name jms_guacamole -d -p 8081:8081 -e JUMPSERVER_SERVER=http://$Server_IP:8080 -e BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN jumpserver/jms_guacamole:1.4.6 \
          && echo -e "\033[31m 你的数据库密码是 $DB_PASSWORD \033[0m" \
          && echo -e "\033[31m 你的BOOTSTRAP_TOKEN是 $BOOTSTRAP_TOKEN \033[0m" \
          && echo -e "\033[31m 你的服务器IP是 $Server_IP \033[0m"
# 三、启动和关闭脚本
* 启动脚本
        
        $ vi /opt/start_jms.sh

        #!/bin/bash
        set -e

        export LANG=zh_CN.UTF-8

        # 项目安装位置,默认是/opt
        Project=/opt

        pid=`ps -ef | grep -v grep | egrep '(gunicorn|celery|beat)' | awk '{print $2}'`
        if [ "$pid" != "" ]; then
            echo -e "\033[31m 检测到 Jumpserver 进程未退出,结束中 \033[0m"
            cd /opt && sh stop_jms.sh
            sleep 5s
            pid1=`ps -ef | grep -v grep | egrep '(gunicorn|celery|beat)' | awk '{print $2}'`
            if [ "$pid1" != "" ]; then
                echo -e "\033[31m 检测到 Jumpserver 进程任未退出,强制结束中 \033[0m"
                kill -9 ${pid1}
            fi
        fi

        echo -e "\033[31m 正常启动 Jumpserver ... \033[0m"
        source $Project/py3/bin/activate
        cd $Project/jumpserver && ./jms start -d
        docker start jms_coco
        docker start jms_guacamole

        exit 0

* 关闭脚本

       $ vi /opt/stop_jms.sh

        #!/bin/bash
        set -e

        # 项目安装位置,默认是/opt
        Project=/opt

        docker stop jms_coco
        docker stop jms_guacamole
        source $Project/py3/bin/activate
        cd $Project/jumpserver && ./jms stop

        exit 0
# 四、开机自启
        # 写入 rc.local
        $ chmod +x /etc/rc.local
        $ echo "sh /opt/start_jms.sh" >> /etc/rc.local
