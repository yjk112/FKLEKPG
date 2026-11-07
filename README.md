                                          实验型ELFKK日志采集和可观测监控系统搭建部署（基于rockyLinux8）

            包含内容： filebeat：采集业务（nginx日志），并输出到kafka
           
            kafka：日志消息队列，削谷，数据缓冲
          
            logstash：日志过滤，解析，清选
          
            elasticsearch：日志存储与检索
           
            kibana：日志可视化，实时分析，用于对elasticsearch数据的展示
           
            Prometheus：通过在需要备监控的主机上安装node_exporter，进行对系统，程序的指标进行采集，存储和查询

            grafana： 可视化展示面板，可接入kibana，Prometheus等
            



           主机安排： 
           
           10.0.0.1：  filebeat    nginx    node_exporter    nginx_prometheus_exporter
           
           10.0.0.2:   kafka    node_exporter    kafka_prometheus_exporter 
           
           10.0.0.3:   logstash   node_exporter

           10.0.0.4：    elasticsearch  node_exporter

           10.0.0.5：    kibana   node_exporter

           10.0.0.6：   prometheus   

           10。0.0.7：   grafna
           

          部署流程： （下方每个涉及到的组件的主要配置文件及内容和systemd启动配置都再我的仓库里面，下面我就不写了，节约时间，大部分主要的配置文件我都是放到/opt底下）
          1（10.0.0.1）：浏览器下载filebeat,node_exporter和nginx_prometheus_exporter的Linux版本的tar.gz压缩包（我这里是rockyLinux8版本的），
              下载完成后移动到桌面或者其他磁盘中，使用Linux命令rz上传压缩包或者使用xftp移动压缩包上传（这里我建议用xftp,上传速度很快），
              可能是我的xftp有问题或者是我操作问题传不到我想传到的/opt目录下，默认在/root目录下，使用mv命令移动到/opt目录下，
              分别进行tar -xzvf 命令对这些压缩包进行解压，解压完成后，将解压后的filebeat目录用mv重命名，方便输入，提高容错率，
              然后cd /opt/config目录下，找到filebeat的配置文件，编辑该文件写入配置，保存退出，为了安全性，遵循最小权限原则，创建用户filebeat，
              更改属主属组 chown -R filebeat:filebeat /opt/filebeat，然后cd /etc/systemd/system,touch filebeat.service后再vim编辑输入systemd启动配置，
              使用systemctl daemon-reload重载fileebat配置，或者可以指定配置文件直接运行filebeat，但是我觉得systemd方式在启动和看日志等更为方便，
              所以推荐systemd方式启动开机自启和查看状态，如果systemctl status filebeat 的状态是灰色或者红色，优先看底下error或者红字地方和
              使用journalctl -u filebeat -f 查看日志找出错因，一般都是systemd配置中路径写错,/opt的文件目录没改属主属组或者权限问题，jdk版本的问题
              和systemd配置中所用到的.sh缺失或者权限不够这类问题                  
              
              再到nginx，网上拿到nginx的rpm包，下载下来传到/etc目录下，使用rpm -Uvh安装nginx，rpm -qa | grep nginx检查验证 ，cd /etc/nginx/conf.d,编辑nginx.conf文件写入配置，cd /etc/systemd/system，touch nginx.service,写入配置文件，
              systemctl damon-reload,进行后续操作。          
              
              最后到采集出口node_exporter和专属出口nginx_prometheus_exporter,操作都是一样的。解压到/opt，依旧cd etc/systemd/system 进入新建这两个exporter的启动文件
              继续编辑就行                  

              最后开放端口和临时关闭Linux的安全机制seLinux,firewall-cmd --permanent --add-port=80/tcp --add-port=9100/tcp --add-port=9113/tcp(这里注意个人项目实验环境我直接就这么搞了，
              但是生产环境中，后面两个端口必须添加仅Prometheus放行)，不然会有安全问题，然后firewall-cmd --reload 加载防火墙，保留已连接，最后暂时关闭安全机制setenforce 0
               
              这里10.0.0.1的各组件版本为    filebeat-9.5.4-linux-x86_64.tar.gz    nginx-prometheus-exporter_1.4.0_linux_amd64.tar.gz    
                                         node_exporter-1.12.1.linux-386.tar.gz       nginx-1.24.0-1.el7.ngx.x86_64.rpm
