---
layout: post
title: Hadoop install
parent: Hadoop
---
<details open markdown="block">
  <summary>
    목차
  </summary>
  {: .text-delta }
{:toc}
</details>


> 하둡 설치 문서이며 현재는 스크립트 기반
> 추후 설명 기입 예정

----
   
# Hadoop 설치
Hadoop **Cluster**로 설치   
[Hadoop DOCS](https://hadoop.apache.org/docs/r3.2.3/hadoop-project-dist/hadoop-common/ClusterSetup.html)


```bash
cat <<EOF | tee -a ./hadoop_install.sh
#!/bin/bash
yum update -y
yum install epel-release -y
yum install wget tcping which perl nc -y

yum install java-1.8.0-openjdk -y

cat <<EOFF | tee -a /etc/profile

# JAVA 
JAVA_HOME="\$(dirname "\$(dirname "\$(readlink -f "\$(which java)")")")"
PATH=\$PATH:\$JAVA_HOME/bin

# Hadoop
HADOOP_VERSION=3.2.3
HADOOP_URL=http://mirror.apache-kr.org/hadoop/common/hadoop-\$HADOOP_VERSION/hadoop-\$HADOOP_VERSION.tar.gz
EOFF

source /etc/profile

curl -fSL "\$HADOOP_URL" -o /tmp/hadoop.tar.gz

tar -xvf /tmp/hadoop.tar.gz -C /opt
rm -f /tmp/hadoop.tar.gz 

ln -s /opt/hadoop-\$HADOOP_VERSION/etc/hadoop /etc/hadoop
mkdir /opt/hadoop-\$HADOOP_VERSION/logs
mkdir /hadoop-data
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


# Hadoop Configure

# RUN
