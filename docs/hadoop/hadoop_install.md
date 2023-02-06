---
layout: post
title: Hadoop install
parent: Hadoop
---
<details open markdown="block">
  <summary>
    목차
  </summary>
  {: .no_toc .text-delta }

1. TOC
{:toc}
</details>
   
# Hadoop 설치
Hadoop **Cluster**로 설치   
[Hadoop DOCS](https://hadoop.apache.org/docs/r3.2.3/hadoop-project-dist/hadoop-common/ClusterSetup.html)

### 1. java install
Hadoop은 Java 기반 솔루션으로 java 설치가 필요하다.

```bash
# Java install
yum install java-1.8.0-openjdk -y

# Java Environment
cat <<EOF | tee -a /etc/profile

# JAVA environment
JAVA_HOME="\$(dirname "\$(dirname "\$(readlink -f "\$(which java)")")")"
PATH=\$PATH:\$JAVA_HOME/bin
EOF
```
### 2. hadoop install

Hadoop 설치

```bash
cat <<EOF | tee -a /etc/profile

# Hadoop environment
HADOOP_VERSION=3.2.3
HADOOP_URL=http://mirror.apache-kr.org/hadoop/common/hadoop-\$HADOOP_VERSION/hadoop-\$HADOOP_VERSION.tar.gz
EOF
# Hadoop Version, URL OS Global Variable
source /etc/profile
# Hadoop Archive file Install
curl -fSL "$HADOOP_URL" -o /tmp/hadoop.tar.gz

# Hadoop archive file Extract
tar -xvf /tmp/hadoop.tar.gz -C /opt

# temporary archive file remove
rm -f /tmp/hadoop.tar.gz 

# Soft Link Directory(Folder)
mv /opt/hadoop-$HADOOP_VERSION /opt/hadoop
ln -s /opt/hadoop/etc/hadoop /etc/hadoop
mkdir /opt/hadoop/logs
# Data-Node 일 경우 아래 경로 생성
mkdir /hadoop-data
```

### 3. hadoop os user 

Hadoop OS User 설정

```bash
groupadd -g 1000 -r hadoop
useradd -d /opt/hadoop -g hadoop -r -u 1000 hadoop

chown -R hadoop.hadoop /opt/hadoop
chown -R hadoop.hadoop /hadoop-data

chmod -R 750 /opt/hadoop
chmod -R 740 /hadoop-data
```

### 4. file permission

파일 권한 설정

```bash
chown -R hadoop.hadoop /opt/hadoop
chown -R hadoop.hadoop /hadoop-data

chmod -R 750 /opt/hadoop
chmod -R 740 /hadoop-data
```

### 5. Total Script
아래는 전체 스크립트

```bash
cat <<EOF | tee -a ./hadoop_install.sh
#!/bin/bash

# Package lastest update
yum update -y

# Java install
yum install java-1.8.0-openjdk -y

# OS environment 
cat <<EOFF | tee -a /etc/profile

# JAVA environment
JAVA_HOME="\$(dirname "\$(dirname "\$(readlink -f "\$(which java)")")")"
PATH=\$PATH:\$JAVA_HOME/bin

# Hadoop environment
HADOOP_VERSION=3.2.3
HADOOP_URL=http://mirror.apache-kr.org/hadoop/common/hadoop-\$HADOOP_VERSION/hadoop-\$HADOOP_VERSION.tar.gz
EOFF

# enable enviornment
source /etc/profile

# Install hadoop tar archive file
curl -fSL "\$HADOOP_URL" -o /tmp/hadoop.tar.gz

# Extract hadoop files
tar -xvf /tmp/hadoop.tar.gz -C /opt

# temporary archive file remove
rm -f /tmp/hadoop.tar.gz 

mv /opt/hadoop-$HADOOP_VERSION /opt/hadoop
ln -s /opt/hadoop/etc/hadoop /etc/hadoop
mkdir /opt/hadoop/logs
mkdir /hadoop-data

groupadd -g 1000 -r hadoop
useradd -d /opt/hadoop -g hadoop -r -u 1000 hadoop

chown -R hadoop.hadoop /opt/hadoop
chown -R hadoop.hadoop /hadoop-data

chmod -R 750 /opt/hadoop
chmod -R 740 /hadoop-data
EOF

sed 's/EOFF/EOF/' ./hadoop_install.sh

chmod +x hadoop_install.sh
sh hadoop_install.sh
```

## 문제가 되는 java version
Hadoop Version에 따라 확인 필요한 사항   
> [Hadoop Java DOCS](https://cwiki.apache.org/confluence/display/HADOOP/Hadoop+Java+Versions)
> [JAVA RPM Version DOCS](https://rpmfind.net/linux/rpm2html/search.php?query=java-1.8.0-openjdk)

   
1. 1.8.0_242
2. 1.8.0_191
3. 1.8.0_171

# Hadoop Main node, Sub node SSH Configuration
> SSH 설정(패스워드 없이 접근): Namenode -> Datanode, Namenode <-> Secondary Namenode

DNS Configure

```bash
cat <<EOF | tee -a /etc/hosts
192.168.56.201   hadoop01
192.168.56.202   hadoop02
192.168.56.203   hadoop03
EOF
```

SSH Create

```bash
# SSH Configure
su - hadoop
ssh-keygen -b 2048 -t rsa -f /opt/hadoop/.ssh/id_rsa -q -N ""
```

# Hadoop Configure

# RUN
