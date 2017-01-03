# mod_cluster demonstration

Configured with 2 EAP and 2 apache, communication with Advertise enabled in apache such that any new apache will automatically broadcast itself to the JBoss farm and immediately pick up workers.

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
    
3. Update EAP to listen for the apache advertise feature.

    CLI:
    ```
    
    ```
    XML:
    ```
    <mod-cluster-config advertise-socket="modcluster" balancer="mybalancer" advertise="true" connector="ajp">

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
        ManagerBalancerName mybalancer
        EnableMCPMReceive
        ServerAdvertise On
        AdvertiseFrequency 5
        AdvertiseGroup 224.0.1.105:23364
      
        <Location /mod_cluster-manager>
            SetHandler mod_cluster-manager
            Order deny,allow
            Deny from all
            Allow from 127.0.0.1
            Allow from localhost
        </Location>
      
    </VirtualHost>
    
    ```
5. Start commands. Replace BASE_DIR as needed. These start commands require the server to be bound to the public interface (-Djboss.bind.address=0.0.0.0 binds to all available addresses) on which it will receive the Advertise from the apache servers.  On the same host, it will be the public interface.  On separate hosts, it will be the public host address of the jboss instance that is sharing the same subnet as the apache host.
    ```
    /opt/share/eap6-load-demos/mc_demo_adv/eap-home/bin/standalone.sh -Djboss.node.name=eap6a -c standalone-ha.xml -Djboss.server.base.dir=/opt/share/eap6-load-demos/mc_demo_adv/eap-1 -Djboss.home.dir=/opt/share/eap6-load-demos/mc_demo_adv/eap-home -Djboss.socket.binding.port-offset=0 -Djboss.bind.address=0.0.0.0 
    ```
    ```
    /opt/share/eap6-load-demos/mc_demo_adv/eap-home/bin/standalone.sh -Djboss.node.name=eap6b -c standalone-ha.xml -Djboss.server.base.dir=/opt/share/eap6-load-demos/mc_demo_adv/eap-2 -Djboss.home.dir=/opt/share/eap6-load-demos/mc_demo_adv/eap-home -Djboss.socket.binding.port-offset=100 -Djboss.bind.address=0.0.0.0
    ```

8. Run the Apache servers. 

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

## Valuable Links

tool for testing if apache is advertising and we are capable of hearing the advertisement:
https://github.com/modcluster/mod_cluster/blob/1.3.x/test/java/Advertize.java