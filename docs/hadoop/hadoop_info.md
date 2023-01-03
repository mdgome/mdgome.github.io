---
layout: post
title: What is Hadoop? 
parent: Hadoop
---
---
<details open markdown="block">
  <summary>
    목차
  </summary>
  {: .text-delta }
{:toc}
</details>   

---

# 1.Hadoop 이란   

GFS(google file system)를 모델로 JAVA로 만들어진 오픈소스
  - GFS: Web 크롤링 정보를 저장하기 위하여 개발된 google 고유의 분산 파일 시스템 (Google 에서 개발된 파일 시스템)
  - HDFS은 GFS의 모델을 기반으로 만들어졌다.

용량이 큰 파일을 읽기 위하여 나온 솔루션 비슷한 솔루션으로는 Windows DFS, NFS, CIFS 등이 있다.

파일을 일정 기준(Block)으로 나눠 여러 대의 서버(node)에 저장하게 된다.   
즉, **분산 처리 파일 시스템**이라는 단어로 하둡을 설명할 수 있다.

분산 처리 시스템을 조금 더 풀어서 말해보면 아래와 같다.
- 분산(여러대), 처리(Action), 파일 시스템(데이터 저장) 
- 여러 대에서 처리하며, 데이터를 저장한다
    

## 1-1. Hadoop Version 
상위 버전에서는 하위 버전에서의 단점을 보안하기 위하여 개발이 되므로 Hadoop에 대하여 이해하는데 더 수월할 것이다.   


### Version 1
 - 파일 저장: HDFS   
 - 분산 처리: Map-Reduce     

### Version 2   
 - 파일 저장: HDFS
 - 분산 처리: YARN, HDFS
 - YARN(Job Tracker 대체)
### Version 3
 - 파일 저장 : HDFS
 - 분산 처리 : YARN, HDFS
 - Secondary Name Node   
    - Name Node Latest Snapshot   
 - JAVA Version 1.8

## 1-2. HDFS

> Hadoop Distributed FileSystem   

HDFS는 실시간이 아닌 Batch 목적으로 설계되었다.   
빠른 처리가 필요한 작업은 HDFS는 고려하지 않아야 한다.   

1. Block
    - 파일을 Block 단위로 쪼개어 저장함. Default 64M
2. Replaction
    - 같은 데이터를 다른 노드에 저장한다. (0번 서버의 0번 영역이 1번 서버에도 존재)
    - 아래 그림은 공식 홈페이지에서 발췌한 것이다.
    - ![Untitled](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/images/hdfsdatanodes.png)
3. Main/Sub (Master / Slave)
    - Main(Name node)  
        - Data Block 의 Meta 정보 저장   
        - Data Node 관리   
    - Sub(Data node)
        - Data 저장
    - 아래 그림은 공식 홈페이지에서 발췌한 것이다.
    - ![Untitled](https://hadoop.apache.org/docs/r3.2.3/hadoop-project-dist/hadoop-hdfs/images/hdfsarchitecture.png) 

## 1-3. Map-Reduce
     HDFS의 데이터를 읽어와 처리하는 방식   
1. Map의 경우 데이터를 분산하는 로직이 들어간다.   
2. Reduce는 분산된 데이터를 처리하는 로직이 들어간다.   

Version 1 에서의 Map-Reduce는 아래와 같은 내용이 존재한다.

- Job Tracker(Name node)   
    - 자원 관리 (Resource Manage)   
    - 전체 진행 상황 관리 (Action Manage)   
- Task Traker(Data node)   
    - 작업 수행/처리(Action) 

## 1-4. Secondary Name Node
HDFS에서 Client 에서 작업(Action OR File/Directory Create/Move etc...)을 수행할 경우 Edit Log에 기록하게 된다.   
- fsimage의 최신화 **Edit Log(≒Trasaction Log) Merge** 
            - 많은 수의 파일 생성/삭제, 디렉토리 삭제/생성 등의 Log


## 1-5. YARN
> Yet Another Resource Negotiator
> Cluster Resource Management

### 1-5-1. Resource Manager
### 1-5-2. Node Manager