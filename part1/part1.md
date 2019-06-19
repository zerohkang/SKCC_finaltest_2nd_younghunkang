#09231 강영훈

## AWS Instance 접속
<ul>
 <li> Git bash 접속하여 AWS 접속을 위한 key파일 디렉토리로 이동 [skcc.pem] </li>
 <li> AWS CentOs Instance 접속계정명 : centos </li>
</ul>

```
$ ssh -i skcc.pem centos@13.124.227.184
```

## CentOS 7 관련
<ul>
 <li> service -> systemctl </li>
 <li> chkconfig -> systemctl </li>
 <li> iptables -> firewalld </li>
</ul>

```
<CentOS 6>
$ service cloudera-scm-server start  //cloudera-scm-server 서비스 시작
$ service cloudera-scm-server stop   //cloudera-scm-server 서비스 종료
$ service cloudera-scm-server status //cloudera-scm-server 서비스 상태
$ chkconfig cloudera-scm-server on   //서버 부팅 시 자동실행 설정
$ chkconfig cloudera-scm-server off  //서버 부팅 시 자동실행 해제

<CentOS 7>
$ systemctl start cloudera-scm-server  //cloudera-scm-server 서비스 시작
$ systemctl stop cloudera-scm-server   //cloudera-scm-server 서비스 종료
$ systemctl status cloudera-scm-server //cloudera-scm-server 서비스 상태 
$ systemctl enable cloudera-scm-server //서버 부팅 시 자동실행 설정
$ systemctl disable cloudera-scm-server//서버 부팅 시 자동실행 해제
$ systemctl is-enable 서비스명          //서비스가 자동실행 설정상태인지 확인
$ systemctl list-unit-files --type=service //자동실행 설정된 서비스 리스트 
```

## Pre-Install Step

### 1. yum update

```
$ sudo yum update
$ sudo yum install -y wget
```

### 2. firewall 정지 [방화벽 정지, CentOs 7부터 iptables -> firewalld 변경]
** 대상 : Cluster 전체 Host **
<ul>
 <li> stop : 현재 작동중인 firewall 서비스 종료 </li>
 <li> disable : OS 부팅 시 firewall 자동실행 해제 </li>
</ul>

```
$ systemctl stop firewalld
$ systemctl disable firewalld
```

### 3. Selinux 정지 [보안 프로그램]
** 대상 : Cluster 전체 Host **
<ul>
 <li> sestatus : Selinux 동작모드 확인 [Default : enforcing] </li>
 <li> /etc/selinux/config 파일을 수정하여 상태를 변경 [enforcing -> disabled] </li>
</ul>

```
$ sestatus
$ sudo  vi /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
# enforcing - SELinux security policy is enforced.
# permissive - SELinux prints warnings instead of enforcing.
# disabled - SELinux is fully disabled.
SELINUX=disabled
# SELINUXTYPE= type of policy in use. Possible values are:
# targeted - Only targeted network daemons are protected.
# strict - Full SELinux protection.
SELINUXTYPE=targeted

$ sudo reboot
```

### 4. NTP 설정 [Cluster host 시간 동기화]
** 대상 : Cluster 전체 Host **
<ul>
 <li> NTP 설치 </li>
 <li> NTP 서버 설정 [/etc/ntp.conf 파일에서 기본설정 서버를 주석처리 후 한국의 ntp서버 추가]</li>
 <li> NTP 서비스 시작</li>
 <li> NTP 서비스 시작프로그램에 등록</li>
 <li> NTP 서비스 작동여부 확인</li>
</ul>

```
$ yum install -y ntp

$ sudo vi /etc/ntp.conf
# Use public servers from the pool.ntp.org project. 
# Please consider joining the pool (http://www.pool.ntp.org/join.html). 
#server 0.centos.pool.ntp.org 
#server 1.centos.pool.ntp.org 
#server 2.centos.pool.ntp.org
server kr.pool.ntp.org 
server time.bora.net
server time.kornet.net

$ systemctl start ntpd
$ systemctl enable ntpd
$ ntpq -p
```

### 5. Passwordless SSH connection setting

#### 1. local에서 master 및 slave 서버 ssh로 붙기
-- skcc.pem 파일이 있는 디렉토리로 이동. 
-- $sudo ssh -i skcc.pem 계정(centos)@ip  로 연결

#### 2. 모든 서버의 root계정 passwd 설정.

```
$passwd root //root계정의 비밀번호 설정하기
```

#### 3. 모든 서버에서 host 파일에 서버정보입력
-- /etc로 이동.
```
sudo vi etc/hosts

private ip1 master
private ip2 slave1
private ip3 slave2
private ip4 slave3
private ip5 slave4
```

#### 4. 모든 서버에서 서버의 실제 Hostname 변경

```
$ hostname    //현재 세팅된 서버의 호스트명 출력
$ hostnamectl set-hostname master //호스트명을 master로 설정
```

#### 5. 모든 서버에서 sshd_config 파일 편집

```
$ sudo vi /etc/ssh/sshd_config 
```
-- PasswordAuthentication yes로 설정 및 저장.


#### 6. 설정 후 서비스 restart

```
$ sudo service sshd restart
```

#### 7. (master노드만) ssh 용 public key 생성하기     

keygen의 결과로 2개의 파일이 새로 생긴 것을 알 수 있다.
``` 
$ssh-keygen [이후에 입력창에는 엔터만 계속 누르면 된다.]
```

#### 8. 1~5번까지 각 노드에서 완료 후 master노드에서 slave노드로 public key 정보 복사하기.

```
$ cd ~/.ssh 이동
$ ssh-copy-id root@slave1 [ssh-copy-id username@remote_host]
```
-- permission denied 문제가 발생하면 앞의 config 파일 및 서비스 재시작 여부 확인
   [6번 과정에서의 오류가 있었을 수 있기 때문에, 오류가 나는 호스트에 접속하여 확인]
   
   
#### 9.  master root계정에서 ssh로 slave에 접속 [설정여부 확인]

```
$ ssh slave1
```

### 6. 추가적인 설정 [성능이슈]
[참고]https://www.cloudera.com/documentation/enterprise/latest/topics/cdh_admin_performance.html

#### 6-1. Disable the tuned Service [시스템 모니터링 및 시스템 설정에 대한 동적튜닝을 제공하는 데몬]

```
$ systemctl start tuned        // Ensure that the tuned service is started
$ tuned-adm off                // Turn the tuned service off
$ tuned-adm list               // Ensure that there are no active profiles
-> No current active profile   // The output should contain the following line

$ systemctl stop tuned         // shutdown the service
$ systemctl disable tuned      // disable the service
```

#### 6-2. Disabling Transparent Hugepages

```
sudo vi /etc/rc.d/rc.local
	add -> 
echo "never" > /sys/kernel/mm/transparent_hugepage/enabled
echo "never" > /sys/kernel/mm/transparent_hugepage/defrag

sudo chmod +x /etc/rc.d/rc.local
sudo vi /etc/default/grub
   add -> transparent_hugepage=never (on line GRUB_CMDLINE_LINUX )
grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### 6-3/ Setting the vm.swappiness Linux Kernel Parameter

vm.swappiness[0~100] 값이 클수록 inactive process들에 대한 메모리 스왑이 빈번히 발생
Hadoop Cluster 사용에는 스왑이 적게 발생하도록 세팅하는 것이 적합 [vm.swappiness 1]
```
$ cat /proc/sys/vm/swappiness      // 현재 세팅값 확인
$ sudo sysctl -w vm.swappiness=1   // 스왑값 
```


## CDH install
[참고]https://www.cloudera.com/documentation/enterprise/5-15-x/topics/install_cm_server.html

### Configure a repository for cloudera manager

1. Download the cloudera-manager.repo file for your OS version to the /etc/yum.repos.d/ directory on the Cloudera Manager Server host [CDH가 아니라 cloudera-manager를 다운받아야 함]
```
$ sudo wget https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/cloudera-manager.repo -P /etc/yum.repos.d/
```

2. change the baseurl within cloudera-manager.repo to fit the version you want to install
```
$ cd /etc/yum.repos.d/cloudera-manager.repo

<AS-IS>
# Packages for Cloudera Manager, Version 5, on RedHat or CentOS 7 x86_64         
name=Cloudera Manager
baseurl=https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5/
gpgkey =https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/RPM-GPG-KEY-cloudera 
gpgcheck = 1

<TO-BE>
# Packages for Cloudera Manager, Version 5, on RedHat or CentOS 7 x86_64         
name=Cloudera Manager
baseurl=https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5.15.2/
gpgkey =https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/RPM-GPG-KEY-cloudera 
gpgcheck = 1
```

2. Import the repository signing GPG key [RHEL 7]
```
$sudo rpm --import https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/RPM-GPG-KEY-cloudera
```

### Install JDK on all hosts

1. 설치가능한 자바 리스트 확인
```
yum list java*jdk-devel
```

2. 1.7버전의 JDK 설치 
```
yum install java-1.7.0-openjdk-devel.x86_64
```

3. 설치확인
```
[centos@ip-172-31-46-77 ~]$ java -version
java version "1.7.0_221"
OpenJDK Runtime Environment (rhel-2.6.18.0.el7_6-x86_64 u221-b02)
OpenJDK 64-Bit Server VM (build 24.221-b02, mixed mode)
[centos@ip-172-31-46-77 ~]$ javac -version
javac 1.7.0_221
```

### Install Cloudera Manager Packages [수동버전]

1.cloudera manager를 설치할 호스트에서 아래의 명령어 실행
```
$sudo yum install cloudera-manager-daemons cloudera-manager-server
```

2.Cluster 모든 호스트에 대해 cloudera agent 설치
```
$sudo yum install cloudera-manager-daemons cloudera-manager-agent
```

### [참고] Cloudera-manager 설치 [자동버전]
```
1.  $ wget http://archive.cloudera.com/cm5/installer/5.15.2/cloudera-manager-installer.bin
```
> a. 개인정보 입력 후 다운로드 URL확인  
> b. 5.15.0 설치파일로 설치하니까 5.16버전이 깔렸음  
> c. 설치제거 후 재시도  $ sudo /usr/share/cmf/uninstall-cloudera-manager.sh

```
2.  $ wget http://archive.cloudera.com/cm5/installer/5.15.0/cloudera-manager-installer.bin
```  
> a. 현재 디렉토리에 설치 파일 다운로드 됨

```
3.  $ chmod u+x cloudera-manager-installer.bin
```  
> a. 설치파일의 권한 변경

```
4.  $ sudo ./cloudera-manager-installer.bin
```  
> a. 설치 실행

5. 설치 완료 후 접속정보 안내가 뜨고, 접속이 안될경우 iptable 같은 리눅스 방화벽 확인이 필요하다는 메시지 출력
> a. 접속정보 : 13.124.227.184:7180 [ admin / admin]


### Install Database  [MariaDB 설치]
[참고]https://www.cloudera.com/documentation/enterprise/5-15-x/topics/cm_ig_installing_configuring_dbs.html

<ol>
<li>MariaDB를 사용하여 RDB가 필요한 서비스들을 커버</li>
<li>JDBC Connector 설치필요</li>
<li>필요한 계정 생성</li> 
</ol>


#### 1. Installing MariaDB Server

```
$ sudo yum install mariadb-server
```

#### 2. Configuring and Starting the MariaDB Server [root / admin]

secure_installation 과정을 진행하며 root password 및 보안관련 설정
Default root 비밀번호는 공백
```
$ sudo systemctl enable mariadb
$ sudo systemctl start mariadb
$ sudo /usr/bin/mysql_secure_installation

[...]
Enter current password for root (enter for none):
OK, successfully used password, moving on...
[...]
Set root password? [Y/n] Y
New password:
Re-enter new password:
[...]
Remove anonymous users? [Y/n] Y
[...]
Disallow root login remotely? [Y/n] N
[...]
Remove test database and access to it [Y/n] Y
[...]
Reload privilege tables now? [Y/n] Y
[...]
All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

#### 3. Installing the MySQL JDBC Driver for MariaDB

* MariaDB JDBC driver is not supported. Install and use the MySQL JDBC driver instead.
* Install the JDBC driver on the Cloudera Manager Server host, as well as any other hosts 
  running services that require database access
  
3-1. Download the MySQL JDBC driver from http://www.mysql.com/downloads/connector/j/5.1.html (in .tar.gz format).
Extract the JDBC driver JAR file from the downloaded file

```
$ tar zxvf mysql-connector-java-5.1.46.tar.gz
```

3-2. Copy the JDBC driver, renamed, to /usr/share/java/. If the target directory does not yet exist, 
     create it. For example

```
$ sudo mkdir -p /usr/share/java/
$ cd mysql-connector-java-5.1.46
$ sudo cp mysql-connector-java-5.1.46-bin.jar /usr/share/java/mysql-connector-java.jar
```

#### 4.Creating Databases for Cloudera Software

4-1. Log in as the root user, or another user with privileges to create database and grant privileges:
```
$ mysql -u root -p
```

4-2. Create databases for each service deployed in the cluster 
     Configure all databases to use the utf8 character set.

```
MariaDB > CREATE DATABASE <database> DEFAULT CHARACTER SET <character set> DEFAULT COLLATE utf8_general_ci;
        > Query OK, 1 row affected (0.00 sec)
MariaDB > GRANT ALL ON <database>.* TO '<user>'@'%' IDENTIFIED BY '<password>';
        > Query OK, 1 row affected (0.00 sec)
MariaDB > SHOW DATABASES;  // Confirme that you have created all  Databased

<Table>
CREATE DATABASE scm DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE amon DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE rman DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE hue DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE metastore DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE sentry DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE nav DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE navms DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE oozie DEFAULT CHARACTER SET DEFAULT COLLATE utf8_general_ci;

<계정생성 및 권한부여>
GRANT ALL ON scm.* TO  'scm'@'%' IDENTIFIED BY 'scm';   
GRANT ALL ON amon.* TO  'amon'@'%' IDENTIFIED BY 'amon';   
GRANT ALL ON rman.* TO  'rman'@'%' IDENTIFIED BY 'rman';  
GRANT ALL ON hue.* TO  'hue'@'%' IDENTIFIED BY 'hue';  
GRANT ALL ON metastore.* TO  'hive'@'%' IDENTIFIED BY 'hive';  
GRANT ALL ON sentry.* TO  'sentry'@'%' IDENTIFIED BY 'sentry';  
GRANT ALL ON nav.* TO  'nav'@'%' IDENTIFIED BY 'nav';  
GRANT ALL ON navms.* TO  'navms'@'%' IDENTIFIED BY 'navms';  
GRANT ALL ON oozie.* TO  'oozie'@'%' IDENTIFIED BY 'oozie';  
```

### Set up the Cloudera Manager Database

#### 1.scm_prepare_database.sh 사용하여 configuration 진행  

MariaDB의 경우 세팅할때 mysql 옵션 사용
```
파일명 :  /usr/share/cmf/schema/scm_prepare_database.sh 
실행예 : sudo ./scm_prepare_database.sh mysql scm scm
```

#### 2.cloudera-scm-manager Database 설정 후 재기동 시, 에러없이 서버가 정상적으로 올라감   

```
$ systemctl cloudera-scm-server restart
$ tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log  //로그확인
$ netstat -antp | grep 7180                                     //서버의 default port도 떴다.
```

### Cloudera Manager Web UI : Install CDH and Other Software

#### 1.Specify hosts for your CDH cluster installation


#### 2.Select Repository


#### 3.Accept JDK License
 > a. cloudera manager 호스트에만 JDK를 깔았기 때문에 설치하기로 설정해야 함


#### 4. Single User Mode
 > a. 체크하지 않음


#### 5. Enter Login Credentials
 > a. root 계정 비밀번호를 입력
 > b. root로 접속 / 비밀번호 : admin


#### 6. Install Agents
 > a. 5개 서버의 [Private IP] 정보 5줄을 입력하고 다음으로 넘어가면 검색결과 확인


#### 7. Install Parcels 
 > a. 기본 패키지 설치
  
  
#### 8. Select Services 
 > 아래의 옵션들 중에서 하나를 선택하여 설치 진행 [Cloudera Navigator 설치안함]

	Core Hadoop
	HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, and Hue

	Core with HBase
	HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, Hue, and HBase

	Core with Impala
	HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, Hue, and Impala

	Core with Search
	HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, Hue, and Solr

	Core with Spark
	HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, Hue, and Spark

	All Services
	HDFS, YARN (MapReduce 2 Included), ZooKeeper, Oozie, Hive, Hue, HBase, Impala, Solr, Spark, and Key-Value Store Indexer

	Custom Services
	Choose your own services. Services required by chosen services will automatically be included. Flume can be added 
	after your initial cluster has been set up.


#### 9. Assign Roles 

 > 8번 단계에서 선택한 서비스들의 각 요소들을 어느 서버에 배치할 것인지 선택

 > DB를 사용하는 서비스들의 경우 JDBC Connector가 설치된 곳에 배치해야만 한다.
 
 > RDB를 사용하는 Component List
    [참고]https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_installing_configuring_dbs.html

<ul>
 <li> Oozie Server </li>
 <li> Sqoop Server </li>
 <li> Activity Monitor </li>
 <li> Reports Manager </li>
 <li> Hive Metastore Server </li>
 <li> Hue Server </li>
 <li> Sentry Server </li>
 <li> Cloudera Navigator Audit Server </li>
 <li> Cloudera navigator metadata Server </li>
</ul>
 
 > Recommended Cluster setting
  [참고]https://www.cloudera.com/documentation/enterprise/5-15-x/topics/cm_ig_host_allocations.html#host_role_assignments
 
 ![Alt text](https://github.com/gogohs/skccBigData/blob/master/recommended_set.PNG)
  
 
 > 실습결과
   ![Alt text](https://github.com/gogohs/skccBigData/blob/master/service설정1.PNG)
   ![Alt text](https://github.com/gogohs/skccBigData/blob/master/service설정2.PNG)
   ![Alt text](https://github.com/gogohs/skccBigData/blob/master/service설정3.PNG)
   
   
## 실습환경 세팅
 
### 1. training 계정 생성
 
#### 1-1. Linux 계정생성
```
$ adduser training
$ passwd training
$ usermod -aG wheel training
## 원래는 visudo에서 주석해제 단계가 있는데 우선은 하지말자
```
 
#### 1-2. HDFS 계정생성

<수동생성>
```
$ sudo -u hdfs hdfs dfs -mkdir /user/training 	         //계정생성
$ sudo -u hdfs hdfs dfs -chown training /user/training   //계정소유자 변경

## 참고사항 : HDFS디렉토리의 권한을 변경하고 싶을경우
$ sudo -u hdfs hdfs dfs -chmod 777 /user/training
```

<자동생성>
Hue UI에 처음 접속을 training 계정으로 할 경우, training계정은 자동으로 HUE admin 계정이 되고
자동으로 HDFS 계정도 생성된다.


#### 3-1. MariaDB에 training 게정을 생성

```
$ mysql -u root -p

## 모든 호스트에서 접속하는(%) training계정에, 모든 Database에 대한, 모든 권한 부여
$ GRANT ALL ON *.* TO 'training'@'%' IDENTIFIED BY 'training';
$ FLUSH PRIVILEGES;

## training 계정에 대한 권한을 중복으로 부여하면, 아래의 결과가 2줄이 나오고, 1줄을 지워주면 정상작동
$ select * from mysql.user where user ='training';
+-----------+----------+
| Host      | User     |
+-----------+----------+
| %         | training |
| localhost | training |
+-----------+----------+
```