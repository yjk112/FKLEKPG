                                          实验型ELFKK日志采集和可观测监控系统搭建部署（基于rockyLinux8）

               包含内容： 
             filebeat：采集业务（nginx日志），并输出到kafka           
              
             kafka：日志消息队列，削谷，数据缓冲
          
            logstash：日志过滤，解析，清选
          
            elasticsearch：日志存储与检索
           
            kibana：日志可视化，实时分析，用于对elasticsearch数据的展示
           
            Prometheus：通过在需要备监控的主机上安装node_exporter，进行对系统，程序的指标进行采集，存储和查询

            grafana： 可视化展示面板，可接入kibana，Prometheus等
            



           主机安排： 
           
           10.0.0.1：  filebeat    nginx    node_exporter    nginx_prometheus_exporter
           
           10.0.0.2:   kafka    node_exporter    kafka_prometheus_exporter 
           
           10.0.0.3:   logstash   

           10.0.0.4：    elasticsearch  

           10.0.0.5：    kibana   

           10.0.0.6：   prometheus   

           10。0.0.7：   grafna
           

          部署流程： （下方每个涉及到的组件的主要配置文件及内容和systemd启动配置都再我的仓库里面，下面我就不写了，节约时间，大部分主要的配置文件我都是放到/opt底下）
          1（10.0.0.1）：
           （1.1）   浏览器下载filebeat,node_exporter和nginx_prometheus_exporter的Linux版本的tar.gz压缩包（我这里是rockyLinux8版本的），
              下载完成后移动到桌面或者其他磁盘中，使用Linux命令rz上传压缩包或者使用xftp移动压缩包上传（这里我建议用xftp,上传速度很快），
              可能是我的xftp有问题或者是我操作问题传不到我想传到的/opt目录下，默认在/root目录下，使用mv命令移动到/opt目录下，
              分别进行tar -xzvf 命令对这些压缩包进行解压，解压完成后，将解压后的filebeat目录用mv重命名，方便输入，提高容错率，
              然后cd /opt/config目录下，找到filebeat的配置文件，编辑该文件写入配置，保存退出，为了安全性，遵循最小权限原则，创建用户filebeat，
              更改属主属组 chown -R filebeat:filebeat /opt/filebeat，然后cd /etc/systemd/system,touch filebeat.service后再vim编辑输入systemd启动配置，
              使用systemctl daemon-reload重载fileebat配置，或者可以指定配置文件直接运行filebeat，但是我觉得systemd方式在启动和看日志等更为方便，
              所以推荐systemd方式启动开机自启和查看状态，如果systemctl status filebeat 的状态是灰色或者红色，优先看底下error或者红字地方和
              使用journalctl -u filebeat -f 查看日志找出错因，一般都是systemd配置中路径写错,/opt的文件目录没改属主属组或者权限问题，jdk版本的问题
              和systemd配置中所用到的.sh缺失或者权限不够这类问题                  
              
             （1.2） 再到nginx，网上拿到nginx的rpm包，下载下来传到/etc目录下，使用rpm -Uvh安装nginx，rpm -qa | grep nginx检查验证 ，cd /etc/nginx/conf.d,
              编辑nginx.conf文件写入配置，cd /etc/systemd/system，touch nginx.service,写入配置文件，
              systemctl damon-reload,进行后续操作。          
              
             （1.3） 最后到采集出口node_exporter和专属出口nginx_prometheus_exporter,操作都是一样的。解压到/opt，依旧cd etc/systemd/system 
             进入新建这两个exporter的启动文件，继续编辑就行                  

             （1.4） 最后开放端口和临时关闭Linux的安全机制seLinux,firewall-cmd --permanent --add-port=80/tcp --add-port=9100/tcp --add-port=9113/tcp
             (这里注意个人项目实验环境我直接就这么搞了，但是生产环境中，后面两个端口必须添加仅Prometheus放行)，不然会有安全问题，然后firewall-cmd --reload 加载防火                 墙，保留已连接，最后暂时关闭安全机制setenforce 0
               
             （1.5） 这里10.0.0.1的各组件版本为    filebeat-9.5.4-linux-x86_64.tar.gz    nginx-prometheus-exporter_1.4.0_linux_amd64.tar.gz    
                                         node_exporter-1.12.1.linux-386.tar.gz       nginx-1.24.0-1.el7.ngx.x86_64.rpm









          2.（10.0.0.2）
                    （2.1）  依旧浏览器中下载kafka的tar.gz的压缩安装包，上传到/opt目录上，tar -xzvf 解压。mv 重命名kafka，cd /opt/kafka/config,vim编辑kafka.yml,
                    写入input和output的配置后保存退出，创建kafka用户，为/opt/kafka更改为kafka的属主属组，给/opt/kafka/bin下的所有.sh加执行权限，chmod +x                                /opt/kafka/bin/*.sh(这里我全加了执行权限，一般来说systemd要哪个可执行的.sh文件就给哪个)，cd /etc/systemd/system,touch kafka.service,写                              入配置内容后保存，systemd daemon-reload重载后面正常操作就行。注意我这里的kafka是新版的。不是需要zookeeper的旧版本kafka，新版本kafka自带                              kraft,kafka.yml内写入关于kraft的内容就行。

                        
                      （2.2） node_exporter和kafka_prometheus_exporter和上方nginx的出口部署方式一样，只是配置文件爱你的名字不一样和配置文件里面的内容不一样，看我的                         仓库文件就行这里不多说

                      （2.3）还是开放端口的和暂时关闭selinux, firewall-cmd --permanent --add-port=9092/tcp  --add-port=9308  --add-port=9308/tcp,后面那两个在上面                         nginx那已经说过注意事项了，暂时关闭安全机制selinux,setenforce 0

                      （2,4）最后测试一下10.0.0.1的filebeat和10.0.0.2的kafka，在1上nc -zv 10。0.0.2 9092查看是否连同，如果可以的话，就curl http://10.0.0.1:80,测试                        端 口产生nginx欢迎页的HTML日志多测试几遍，到10.0.0.2进行查看是否有日志进来   从头开始消费，看历史所有消息/opt/kafka/bin/kafka-console-                                consumer.sh --bootstrap-server 127.0.0.1:9092 --topic filebeat-logs --from-beginning


                       （2.5）这里的10.0.0.2的kafka版本是kafka_2.13-4.4.0.tgz




         3.(10.0.0.3)
                     (3.1)   当kafka消息队列中的nginx日志到logstash时。logstash也是一样的安装流程到/opt下解压后，vim /opt/logstash/config/nginx-kafka.conf后，
                     写入  input和output和时区校正，一样的到system哪里穿件service文件，写入启动配置，systemclt daemon-reload重载

                    （3.4）这里10.0.0.3的logstash版本为logstash-9.5.5-linux-x86_64.tar.gz。其次就是这里kafka也可以不用部署的，filebeat可以直接把nginx日志收集完丢给
                    logstash，因为我这就只有nginx的日志产生，数量少且内容简单，kafka还是用在日志种类复杂数量居多的情境下比较适合





         4.（10.0.0.4）
                      （4.1） 当logstash解析过滤清洗后，输出到elasticsearch存储与检索，elasticsearch不需要配置啥，因为logstash输出到elasticsearch，elasticsearch被                       logstash主动推送，然后后面的kibana又是主动拉取查询elasticsearch内存储的日志，所以elasticsearch的输入和输出都是被动的，这里要做的就是
                      systemd配置，开放端口和临时关闭Linux安全机制，但首先是把压缩包解压掉，然后正常配置systemd。systemctl daemon-reload重载，firewall-cmd --                             permannet--add-port=9200/tcp和setenforce 0，还有要注意elaticsearch和kibana的版本要一样。不然有些配置对不上，搞得kibana有些功能用不了


                       （4.2） 这里10.0.0.4的elasticsearch版本为elasticsearch-9.5.5-linux-x86_64.tar.gz 





         5.(10.0.0.5)
                      (5.1)  kibana拉取和查询elasticsearch内的日志，要玩kibana的话先去了解索引和分片的这两个概念，还有进入kibana的web端等待的时间需要长一些，如果                          点击可视化这些功能报错进不去可能是系统索引丢失或锁环的问题，kibana一样的解压一样的kibana.yml配置一样的systemd配置一样的重载，一样的开放
                      端口firewall-cmd --ermanent --add-port=5601/tcp和临时关闭安全机制,然后每个组件的开机顺序就是filebeat——kafka——logstash——elasticsearch——                              kibana

                      5（2）启动完成后进入到kibana客户端，如果拒绝连接或者是其他问题，看端口是否被占用或者是权限问题，又或者是各组件的配置文件问题等等问题又或者是一
                    些很细的问题比如sh脚本的缺失损坏这些，优先从头开始排查错误

                      5（3）最后成功进去后，怎么操作kibana就去看其他技术文档吧，我也不是很熟这个玩意我后面再去多学学，所以elfkk日志采集与可视化这部分搭建完成，我的配                        置文件啥的应该没问题。 

                      （5.4）  这里10.0.0.5的kibana版本为kibana-9.5.5-linux-x86_64.tar.gz



         6.(10.0.0.6)
                     (6.1)   还是解压Prometheus压缩包到/opt , mv重命名,cd  config,这里有三个配置文件，一个是主要配置文件，一个是主动拉取各个node_exporter采集的信息,另一个                         就是主动拉取kafka_prometheus_exporter和nginx_prometheus_exporter采集的信息,文件配置都在仓库中，然后进入system的目录中创建service写入启动配                                    置后保存退出重载


                     (6.2)    开放端口和临时关闭安全机制 ，firewall-cmd  --permanent --add-port=9090和setenforce 0

                    （6.3）        这里10.0.0.6的Prometheus版本为prometheus-3.13.5.linux-386.tar.gz




          7.（10.0.0.7）
                     （7.1） 还是解压grafana压缩包到/opt，mv重命名，这里直接编辑system文件即可，编写完重载，然后开放端口和安全机制问题，firewall-cmd --permannet --                        add -port=3000/tcp 和setenforce 0,进入grafanaweb端，账号密码都为admin，然后添加采集源搜索Prometheus输入Prometheus的URL然后确认添加后完成，最后                     简单的可观测监控基本完成，这里面有些自定义玩法或者其他我就不说了



                      （7.2）这里10.0.0.7的grafana的版本为grafana-12.3.6.tar.gz
                      


         8.总结：这套微型的日志采集与可观测监控系统就搭建部署完成了，文件里面的IP和我这个笔记里的IP不一样，记得改为自己机器上的IP
                 里面更深层的东西建议多看其他优秀的技术文档和相关视频学习，作为一位小白，
                 我这里有些地方的内容可能不小心忽略了，请见谅，总的来说，ELFK 是日志栈，用来采集、清洗、存储、检索原始日志，
                 用 Kibana 查看日志，适合故障回溯、排查具体报错，Prometheus + Grafana 是时序监控体系，通过各类 exporter 采集数值指标，Prometheus 定时拉取存储， Grafana 做监控大屏和告警，适合观测系统负载、提前预警；两个体系互相配合：监控发现异常 → 去日志系统查原始日志定位根因。


                      
