# JVM
Performance Tuning and Trouble shooting

### Basic JVM_Tunning

1. Heap 의 Max 와 Min Size 는 같게
2. PermSize 를 지정할것
3. -server 옵션 지정
4. NewSize, MaxNewSize는 같게, 전체 Heap 의 1/3 크기
5. Full GC Time 이 많이 소요될 경우 CMS( Concurrnet GC )사용 : Tomcat 7 Default
6. 32Bit JVM의 경우 HeapSize는 Max 1200M~1500M 까지
7. Full GC는 통상적으로 1시간에 1회, 1회 5~12 초가 적절
8. gc log를 남겨 gc 트렌드 분석,JVM 튜닝 후 제거

~~~
-server 
-Xms2g -Xmx2g
-XX:MaxPermSize=256m -XX:+UseCompressedOops
-XX:NewSize=512m -XX:MaxNewSize=512m
-XX:+UseConcMarkSweepGC -XX:+UseParNewGC    
-XX:+CMSPermGenSweepingEnabled -XX:+CMSClassUnloadingEnabled
-XX:+CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=50 -XX:+UseCMSInitiatingOccupancyOnly  
-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps
-Xloggc:/logs/gc.log
-XX:+HeapDumpOnOutOfMemoryError
~~~

WAS의 성능에 큰 영향을 주는 것 중 하나가 JVM이다. JVM의 튜닝 여부에 따라서 WAS에서 작동하는 애플리케이션의 성능을 크게는 20~30%까지 향상시킬 수 있다. Slow down과 Hang up을 일으키는 직접적인 요인이 되는 것은 JVM의 Full GC이다. 

Native Heap ( C-Heap ) 영역은 Thread, Stack, code cache 로 사용된다.

Java Heap 은 Java 프로그램 클래스와 인스턴스 array 가 저장되는 주용 영역으로, 크게 New(eden+S0+S1 또는 Yuong) 영역, Old (Tenered) 영역 그리고 Perm 영역 3가지로 분류된다. 

Perm 영역은 클래스나 메쏘드들이 로딩되는 영역으로 성능상의 영향은 거의 미치지 않는다. Java8 부터 Perm 영역은 Metaspace 로 변경되어 IBM JDK 와 유사하게 Native Memory를 사용한다.

주목해야 할 부분은 객체의 생성과 저장에 관련되는 New와 Old 영역인데, 모든 객체는 생성되자마자 New 영역에 저장되고 시간이 지남에 따라 이 객체들은 Old 영역으로 이동된다. New 영역을 클리어하는 과정을 Minor GC라 하고 Old 영역을 클리어하는 과정은 Major GC 또는 Full GC라 하는데, 성능상의 문제는 이 Full 영역에서 발생한다. 

Minor GC의 경우는 1초 이내의 고속으로 이뤄지는 작업이기에 신경쓸 필요가 없지만, Full GC의 경우에는 시간이 매우 오래 걸린다. 또한 Full GC가 발생할 동안은 애플리케이션이 순간적으로 멈춰버리기 때문에 시스템이 순간적으로 Hang up으로 보이거나 Full GC가 끝나면서 갑자기 요청이 몰려버리는 현상 때문에 종종 시스템의 장애를 발생시키는 경우가 있다. 

Full GC는 통상 1회에 3~5초 정도가 적절하고 보통 하루에 JVM 인스턴스당 5회 이내가 적절하다고 여겨진다. Full GC가 자주 일어나는 것이 문제가 될 경우에는 JVM의 힙(heap) 영역을 늘려주면 천천히 일어나지만 반대로 Full GC에 소요되는 시간이 증가한다. 개당 Full GC 시간이 오래 걸릴 경우에는 JVM의 힙 영역을 줄여주면 빨리 Full GC가 끝나지만 반대로 Full GC가 자주 일어난다는 단점이 있어 이 부분에 대한 적절한 튜닝이 필요하다. 

대부분의 Full GC로 인한 문제는 JVM 자체나 WAS의 문제이기보다는 그 위에서 구성된 애플리케이션이 잘못 구성되어 메모리를 과도하게 사용하거나 오래 점유하는 경우가 있다. 예를 들어 대용량 데이터베이스 WAS 쿼리의 결과를 WAS의 메모리에 보관하거나 세션(session)에 대량의 데이터를 넣는 것들이 대표적인 예가 될 수가 있다.                  

## More Options

###  JDK 1.7  이하
~~~
-server 
-Xms2g -Xmx2g
-XX:MaxPermSize=256m -XX:+UseCompressedOops
-XX:NewSize=512m -XX:MaxNewSize=512m
-XX:+UseConcMarkSweepGC -XX:+UseParNewGC    
-XX:+CMSPermGenSweepingEnabled -XX:+CMSClassUnloadingEnabled
-XX:+CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=50 -XX:+UseCMSInitiatingOccupancyOnly  
-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps
-Xloggc:/logs/gc.log
-XX:+HeapDumpOnOutOfMemoryError
~~~

Note : 1~2개의 코어를 가진 단일 CPU에 CMS 적용시 아래 옵션 추가 
~~~
-XX:+CMSIncrementalMode                      
~~~

### JDK 1.8.x 이상 Hotstop JVM
Note : Perm 영역이 Metaspace 영역으로 변경됨

~~~
-server
-Xms2g -Xmx2g
-XX:MaxMetaspaceSize=128m
-XX:NewSize=512m -XX:MaxNewSize=512m
-XX:+UseConcMarkSweepGC -XX:+UseParNewGC
-XX:+CMSClassUnloadingEnabled
-XX:+CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly
-XX:+ScavengeBeforeFullGC -XX:+CMSScavengeBeforeRemark
-XX:+DisableExplicitGC -Dfile.encoding=UTF-8 -Djna.nosys=true
-Duser.timezone="Asia/Seoul"
-verbose:gc -XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/logs/gc.log
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M
-XX:+HeapDumpOnOutOfMemoryError
-Djava.rmi.server.hostname=%external.ip%
-Dcom.sun.management.jmxremote.port=%my.jmx.port%
-Dcom.sun.management.jmxremote.rmi.port=%my.rmi.port%
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
~~~

Example
~~~
-server -Xms256m -Xmx256m -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSPermGenSweepingEnabled -XX:+CMSClassUnloadingEnabled -XX:+CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:gc.log -XX:+HeapDumpOnOutOfMemoryError -XX:+DisableExplicitGC -Dfile.encoding=UTF-8 -Djna.nosys=true -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=8999 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false
~~~


### G1 GC
Note : 
- Memory 6G 이상 , JDK 1.7.x 이상일 경우 사용 가능
- StringDeduplication 은 JDK 1.8.20 이상부터 적용 가능 ( sting 객제의 중복 제거 기능 )
- Java 9 부터 기본 옵션

~~~
-server
-Xms8g -Xmx8g
-XX:+UseG1GC -XX:MaxGCPauseMillis=200
-XX:+UseStringDeduplication
-XX:+UnlockDiagnosticVMOptions 
-XX:+G1SummarizeConcMark
-XX:InitiatingHeapOccupancyPercent=35
-XX:G1ReservePercent=10
-verbosegc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintAdaptiveSizePolicy
-Xlog:gc*=debug
-Xloggc:/logs/gc.log
-XX:+HeapDumpOnOutOfMemoryError
~~~



### References

http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html

http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html

http://java.dzone.com/articles/java-8-permgen-metaspace

http://blog.sokolenko.me/2014/11/javavm-options-production.html

https://www.dynatrace.com/news/blog/understanding-g1-garbage-collector-java-9/


### ThreadDump

WAS의 행이 의심될 경우에는 thread dump 를 추출하여 쓰레드상의 경합여부 및 속도가 느린 원인을 분석한다.

thread dump는 아래의 명령을 통해 WAS 의 standard output 으로 출력된다. tomcat 의 경우에는 기본으로 catalina.out 파일에 떨어진다.

thread dump는 4~5초 간격으로 3회 수행한다.

~~~
kill -3 <pid>
~~~
