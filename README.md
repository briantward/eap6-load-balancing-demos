# Load Balancing Strategy Demonstrations

These demonstrations are all built on EAP 6.3.0 with EWS 2.1.0 downloaded from access.redhat.com.  

## 1. Building the demo environment base:

1. Extract the EAP server.
    ```
    unzip jboss-eap-6.3.0.zip -d DEMO_FOLDER
    unzip jboss-eap-native-webserver-connectors-6.3.0-RHEL7-x86_64.zip -d DEMO_FOLDER
    mv DEMO_FOLDER/jboss-eap-6.3 DEMO_FOLDER/eap-home
    ```
2. Duplicate the number of standalone servers you need.
    ```
    mv DEMO_FOLDER/eap-home/standalone DEMO_FOLDER/eap-1
    cp -Ra DEMO_FOLDER/eap-1 DEMO_FOLDER/eap-2
    ```

3. Enable the modeler within tomcat to use the mod_cluster demo tool in part 4 below.
    
    CLI:
    ```
    /system-property=org.apache.tomcat.util.ENABLE_MODELER/:add(value=true)
    ```
    
    XML (normally added after \<extensions\>):
    ```
    <system-properties>
       <property name="org.apache.tomcat.util.ENABLE_MODELER" value="true"/>
    </system-properties>
    ```

4. Extract the Apache server. 
    ```
    unzip jboss-ews-httpd-2.1.0-RHEL7-x86_64.zip -d DEMO_FOLDER
    mv DEMO_FOLDER/jboss-ews-2.1 DEMO_FOLDER/ews-1
    cp -Ra DEMO_FOLDER/ews-1 DEMO_FOLDER/ews-2
    ```
    
5. Configure the Apache server with the included scripts.  The script will build absolute paths to resources in the configurations. Remove auth_kerb.conf if you don't plan on making use of it.
    ```
    cd DEMO_FOLDER/ews-1/httpd/
    ./.postinstall
    cd ../../ews-2/httpd/
    ./.postinstall
    cd ../../
    rm ews-1/httpd/conf.d/auth_kerb.conf
    rm ews-2/httpd/conf.d/auth_kerb.conf
    ```
6. Then extract or copy the demo files in this project overtop your completed setup.  See respective demo for configuration explanations.

## 2. Choose a Subfolder Demonstration Example
* mjk_demo - mod_jk configuration
* mc_demo - mod_cluster configured without ServerAdvertise from apache, using proxy-list in EAP to know the apache instances
* mc_demo_adv - mod_cluster configured with ServerAdvertise over UDP
* mc_demo_tcp - mod_cluster configured with jgroups TCP stack (for cache distribution)

Replace the configs in these demos with that of your server built in step 1, or review the documentation in each to do the configuration step-by-step yourself.

N.B. These demos are setup with the minimum necessary number of apache and jboss servers to demonstrate load balancing.  2 over 2 is not a specifically recommended architecture.
  
## 3. Deploy cluster example application
cluster.war is based off this repo: 
[https://github.com/akquinet/jbosscc-as7-examples]
   
## 4. Analyze with load balancing tool 
This tool was developed by the mod_cluster team, but it can be used for testing mod_jk as well.
1. Deploy the load-demo.war to the EAP instances.
2. Run the client from your local or other remote machine.  It requires a java GUI interface.

N.B. This tool will run from the host you use it on, thus any requests from it come from the same IP.  If you are sending traffic directly to a hardware-based load balancer in front of the apache software-based load balancer, then you will probably hit the same apache software-based load balancer every time.  That behavior is governed by the hardware load balancer.  
   
## Valuable links

Support offerings:

* [Does Red Hat / JBoss offer support for Apache, mod_jk, mod_proxy, or mod_cluster?](https://access.redhat.com/solutions/18495)
* [JBoss Enterprise Application Platform (EAP) 6 Supported Configurations](https://access.redhat.com/articles/111663)
* [JBoss Enterprise Application Platform Component Details](https://access.redhat.com/articles/112673) Details the specific versions of components based on EAP version.

Load Calculations & Monitoring:
* [Load Balancer Configuration Tool](https://access.redhat.com/labs/lbconfig)
* [ab - Apache HTTP server benchmarking tool](https://httpd.apache.org/docs/2.2/programs/ab.html)
* [Apache Module mod_status](https://httpd.apache.org/docs/2.2/mod/mod_status.html)
* [How to monitor Apache httpd process/thread status](https://access.redhat.com/solutions/39808) Essentially tells you to enable mod_status
* [How to connect to JBoss EAP 6 using JConsole - Red Hat Customer Portal](https://access.redhat.com/solutions/149973)
* [11.8.7. Disable Remote Access to the JMX Subsystem](https://access.redhat.com/documentation/en-US/JBoss_Enterprise_Application_Platform/6.3/html/Administration_and_Configuration_Guide/Disable_Remote_Access_to_the_JMX_Subsystem1.html)

Configuration Instructions:
* [EAP 6.3 Administration and Configuration Guide: Chapter 19. HTTP Clustering and Load Balancing](https://access.redhat.com/documentation/en-US/JBoss_Enterprise_Application_Platform/6.3/html/Administration_and_Configuration_Guide/chap-HTTP_Clustering_and_Load_Balancing.html)
** Please be sure to use the guide for the version you are running.  Chapter numbers do not stay the same in each guide.
* [19.3.6. Configure JBoss EAP 6 to Accept Requests From External Web Servers](https://access.redhat.com/documentation/en-US/JBoss_Enterprise_Application_Platform/6.3/html/Administration_and_Configuration_Guide/Configure_the_Enterprise_Application_Platform_to_Accept_Requests_From_an_External_HTTPD1.html)
* [19.5.2. Configure the mod_cluster Subsystem](https://access.redhat.com/documentation/en-US/JBoss_Enterprise_Application_Platform/6.3/html/Administration_and_Configuration_Guide/Configure_the_mod_cluster_Subsystem.html)
* [2016 - Configuring a JBoss EAP 7 Cluster](https://access.redhat.com/articles/2359241)
* [2013 - JBoss EAP 6 Clustering](https://access.redhat.com/articles/524633)
* [19.3.6. Configure JBoss EAP 6 to Accept Requests From External Web Servers](https://access.redhat.com/documentation/en-US/JBoss_Enterprise_Application_Platform/6.3/html/Administration_and_Configuration_Guide/Configure_the_Enterprise_Application_Platform_to_Accept_Requests_From_an_External_HTTPD1.html)
* [How to set web connector timeout (connectionTimeout) in EAP 6 - Red Hat Customer Portal](https://access.redhat.com/solutions/456733)
* [How to configure max-connections attribute of HTTP/AJP connector via CLI in JBoss EAP 6.x? - Red Hat Customer Portal](https://access.redhat.com/solutions/1466693)
* [What is the default value of maxThreads for HTTP/AJP Connector in JBoss EAP or Tomcat? - Red Hat Customer Portal](https://access.redhat.com/solutions/25054)
* [JBoss Web - System Properties](http://docs.jboss.org/jbossweb/7.0.x/sysprops.html)
* [Set up SSL between Apache, mod_cluster and JBoss EAP6 - Red Hat Customer Portal](https://access.redhat.com/solutions/199493)
* [mod_proxy_ajp configuration with JBoss EAP6 with SSL enable in Apache - Red Hat Customer Portal](https://access.redhat.com/solutions/193563)
* [JBoss Web Server - xPaaS Middleware Images | Using Images | OpenShift Enterprise 3.0](https://docs.openshift.com/enterprise/3.0/using_images/xpaas_images/jws.html)

Mod_cluster
* [19.5.2. Configure the mod_cluster Subsystem](https://access.redhat.com/documentation/en-US/JBoss_Enterprise_Application_Platform/6.3/html/Administration_and_Configuration_Guide/Configure_the_mod_cluster_Subsystem.html) EAP configuration properties
* [19.3.4. mod_cluster Configuration on httpd](https://access.redhat.com/documentation/en-US/JBoss_Enterprise_Application_Platform/6.3/html/Administration_and_Configuration_Guide/mod_cluster_Configuration_on_httpd.html) Apache configuration properties
* [mod_cluster community documentation](https://docs.jboss.org/mod_cluster/1.2.0/html/)
* [Troubleshooting and Optimizing mod_cluster](https://access.redhat.com/solutions/73033)

Mod_jk
* [In mod_jk why am I seeing the log message "unrecoverable error 200, request failed. Client failed in the middle of request, we can't recover to another instance" ? - Red Hat Customer Portal](https://access.redhat.com/solutions/25314)
* [54621 – [PATCH] custom mod_jk availability checks](https://bz.apache.org/bugzilla/show_bug.cgi?id=54621)
* [[Apache-SVN] Index of /tomcat/jk/trunk](http://svn.apache.org/viewvc/tomcat/jk/trunk/)
* [mod_jk community documentiation](http://tomcat.apache.org/connectors-doc/)

Migration help:
* [mod_jk to mod_cluster parameter mappings](https://docs.jboss.org/mod_cluster/1.2.0/html/mod_jk.html)
* [mod_proxy to mod_cluster parameter mappings](https://docs.jboss.org/mod_cluster/1.2.0/html/mod_proxy.html)
* [Equivalent HTTP/HTTPS/AJP connector attributes mapping between JBoss EAP 5.x and JBoss EAP 6.x - Red Hat Customer Portal](https://access.redhat.com/solutions/110003)
