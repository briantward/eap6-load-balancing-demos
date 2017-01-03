# mod_cluster demonstration

Configured with 2 EAP and 2 apache, communication without Advertise enabled in apache.
Instead proxy-list in EAP points to the apache.

N.B. If you deploy the cluster.war example, the EAP nodes are running a true clustered app in ha mode.

Use standalone-ha.xml as your base.

1. Follow the instructions in the parent folder for guidelines on EAP and EWS setup.

2. Update EAP with an instance-id on the web subsystem. This is like your jvmRoute with mod_jk.
    
    CLI:
    ```
    /subsystem=web:write-attribute(name=instance-id,value=eap6a)
    ```
    ```
    /subsystem=web:write-attribute(name=instance-id,value=eap6b)
    ```
    XML:
    ```
    <subsystem xmlns="urn:jboss:domain:web:2.1" default-virtual-server="default-host" instance-id="eap6a" native="false">
    ```
    ```
    <subsystem xmlns="urn:jboss:domain:web:2.1" default-virtual-server="default-host" instance-id="eap6b" native="false">
    ```
    
3. Update EAP with proxy-list to point to apache instances.

    CLI:
    ```
    /subsystem=modcluster/mod-cluster-config=configuration:write-attribute(name=proxy-list,value="localhost:6666,localhost:6766")
    /subsystem=modcluster/mod-cluster-config=configuration:write-attribute(name=balancer,value="mybalancer")
    /subsystem=modcluster/mod-cluster-config=configuration:write-attribute(name=advertise,value="false")
    ```
    XML:
    ```
    <mod-cluster-config proxy-list="localhost:6666,localhost:6766" balancer="mycluster" advertise="false" connector="ajp">
    ```
    
4. Update httpd.conf on each apache.  Below is the example for ews-1.  Be sure to adjust the port on ews-2 to be 6766. Also note the apache permissions to Directory and Location tags and adjust to your needs.
  
    Ensure the mod_proxy_balancer.so is commented out or removed.  
    
    ```  
    #LoadModule proxy_balancer_module /opt/share/eap6-load-demos/mc_demo/ews-1/httpd/modules/mod_proxy_balancer.so
    ```

    Add the following mod_cluster details:
    
    ```
    LoadModule proxy_cluster_module modules/mod_proxy_cluster.so
    LoadModule slotmem_module modules/mod_slotmem.so
    LoadModule manager_module modules/mod_manager.so
    LoadModule advertise_module modules/mod_advertise.so  
      
    Listen localhost:6666
    <VirtualHost *:6666>
        <Directory "/">
            Order deny,allow
            Deny from all
            Allow from 127.0.0.1
            Allow from localhost
        </Directory>
      
        AllowDisplay On
        KeepAliveTimeout 60
        MaxKeepAliveRequests 0
        ManagerBalancerName mycluster
        EnableMCPMReceive
        ServerAdvertise Off
      
        <Location /mod_cluster-manager>
            SetHandler mod_cluster-manager
            Order deny,allow
            Deny from all
            Allow from 127.0.0.1
            Allow from localhost
        </Location>
      
    </VirtualHost>
    
    ```
5. Start commands. Replace BASE_DIR as needed.
    ```
    BASE_DIR/eap-home/bin/standalone.sh -Djboss.node.name=eap6a -c standalone-ha.xml -Djboss.server.base.dir=BASE_DIR/mc_demo/eap-1 -Djboss.home.dir=BASE_DIR/mc_demo/eap-home -Djboss.socket.binding.port-offset=0 
    ```
    ```
    BASE_DIR/eap-home/bin/standalone.sh -Djboss.node.name=eap6b -c standalone-ha.xml -Djboss.server.base.dir=BASE_DIR/mc_demo/eap-2 -Djboss.home.dir=BASE_DIR/mc_demo/eap-home -Djboss.socket.binding.port-offset=100
    ```

6. Run the Apache servers. 

    ```
    BASE_DIR/ews-1/httpd/sbin/apachectl start
    BASE_DIR/ews-2/httpd/sbin/apachectl start
    ```

Hit the app:
* <http://localhost:80/cluster>
* <http://localhost:81/cluster>

Hit the mod_cluster-manager:
* <http://localhost:6666/mod_cluster-manager>
* <http://localhost:6766/mod_cluster-manager>

## Notes

* "AllowDisplay On" adds a few details to the mod_cluster-manager page and can be safely removed.  Note that it always says ServerAdvertise is on, even when it is not.
* DO NOT use mod_cluster on port 80! This is a potential security hazard.