# mod_jk demonstration

1. Follow the instructions in the parent folder for guidelines on EAP and EWS setup.

2. Update EAP with an instance-id on the web subsystem. The instance id should match your jvmRoute with mod_jk workers.properties. Do this for all EAP servers and note the identifier names.

    CLI:
    ```
    /subsystem=web:write-attribute(name="instance-id",value="eap6a")
    /subsystem=web/connector=ajp:add(socket-binding=ajp, protocol="AJP/1.3", enabled=true, scheme="http")
    ```
    XML:
   
    Add the web instance-id and the AJP connector to your web subsystem.
    ```
    <subsystem xmlns="urn:jboss:domain:web:2.1" default-virtual-server="default-host" native="false" instance-id="eap6a">
        <connector name="ajp" protocol="AJP/1.3" scheme="http" socket-binding="ajp" max-connections="600"/>
    ```
    
3. Update EAP with a system property to use mod_jk.

    CLI:
    
    ```
    /system-property=UseJK/:add(value=true)
    ```
    
    XML:
    
    Add system property.
    ```    
    <system-properties>
        <property name="org.apache.tomcat.util.ENABLE_MODELER" value="true"/>
        <property name="UseJK" value="true"/>
    </system-properties>
    ```
    
4. Update httpd.conf on each apache.  Make sure the JkShmFile storage is on a local disk.

    ```
    LoadModule jk_module modules/mod_jk.so
     
    JkWorkersFile conf/workers.properties
    JkLogFile logs/mod_jk.log
    JkLogLevel info
    JkLogStampFormat "[%a %b %d %H:%M:%S %Y]"
     
    # For mod_rewrite compatibility, use +ForwardURIProxy (default since 1.2.24)
    JkOptions +ForwardKeySize +ForwardURICompatUnparsed -ForwardDirectories
     
    JkRequestLogFormat "%w %V %T"
     
    JkMountFile conf/uriworkermap.properties
     
    JkShmFile run/jk.shm
     
    JkWatchdogInterval 60
     
    <Location /jkstatus>
        JkMount status
        Order deny,allow
        Deny from all
        Allow from 127.0.0.1
        Allow from localhost
    </Location>
    ```

5. Add workers.properties in the apache conf folder.
    ```
    worker.list=loadbalancer,status
     
    worker.eap6a.port=8009
    worker.eap6a.lbfactor=1
    worker.eap6a.reference=worker.template
    worker.eap6b.port=8109
    worker.eap6b.lbfactor=1
    worker.eap6b.reference=worker.template
    worker.eap6c.port=8209
    worker.eap6c.lbfactor=1
    worker.eap6c.reference=worker.template
     
    worker.loadbalancer.balance_workers=eap6a,eap6b,eap6c
    worker.loadbalancer.type=lb
    worker.loadbalancer.sticky_session=true
     
    worker.status.type=status
     
    worker.template.type=ajp13
    worker.template.host=127.0.0.1
    #worker.template.host=localhost
    worker.template.prefer_ipv6=false
     
    worker.template.socket_timeout=10
     
    worker.template.ping_mode=A
     
    worker.template.prepost_timeout=1000 
    worker.template.reply_timeout=240000
    worker.template.retries=1
    #worker.template.retry_interval=100
    #worker.template.max_reply_timeouts=2
    worker.template.recover_time=120    
    worker.template.connection_pool_timeout=600 
    ```

6. Add uriworkermap.properties in the apache conf folder.  This mapping can also be done by using JkMount directives in the httpd.conf file. The uriworker file allows one to dynamically change the context by editing the file.
    
    ```
    # Map application to balancer
    /cluster=loadbalancer
    /cluster/*=loadbalancer
    /load-demo=loadbalancer
    /load-demo/*=loadbalancer
    ```
    
7. Run the EAP servers. Replace BASE_DIR as needed. To work with the mod_cluster load demo app, use the flag -Djboss.mod_cluster.jvmRoute to name your jvmRoute, same as the name used in workers.properties.

    ```
    BASE_DIR/eap-home/bin/standalone.sh -Djboss.server.base.dir=BASE_DIR/mjk_demo/eap-1 -Djboss.home.dir=BASE_DIR/mjk_demo/eap-home -Djboss.socket.binding.port-offset=0 -Djboss.mod_cluster.jvmRoute=eap6a
    ```
    ```
    BASE_DIR/eap-home/bin/standalone.sh -Djboss.server.base.dir=BASE_DIR/mjk_demo/eap-1 -Djboss.home.dir=BASE_DIR/mjk_demo/eap-home -Djboss.socket.binding.port-offset=100 -Djboss.mod_cluster.jvmRoute=eap6b
    ```
    ```
    BASE_DIR/eap-home/bin/standalone.sh -Djboss.server.base.dir=BASE_DIR/mjk_demo/eap-1 -Djboss.home.dir=BASE_DIR/mjk_demo/eap-home -Djboss.socket.binding.port-offset=200 -Djboss.mod_cluster.jvmRoute=eap6c
    ```
8. Run the Apache servers. 

    ```
    BASE_DIR/ews-1/httpd/sbin/apachectl start
    BASE_DIR/ews-2/httpd/sbin/apachectl start
    ```

Hit the app:
* <http://localhost:80/cluster>
* <http://localhost:81/cluster>

Hit the jk status:
* <http://localhost:80/jkstatus>
* <http://localhost:81/jkstatus>
