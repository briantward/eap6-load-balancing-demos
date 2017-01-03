# mod_cluster demonstration

Configured with 2 EAP and 2 apache, communication without Advertise enabled in apache. Instead proxy-list in EAP points to the apache. JGroups clustering uses a true-tcp complete stack with no UDP communication.

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
    <mod-cluster-config proxy-list="localhost:6666,localhost:6766" balancer="mybalancer" advertise="false" connector="ajp">
    ```
    
4. Update EAP with jgroups true-tcp stack

    CLI:
    ```
    batch
    /subsystem=jgroups/stack=true-tcp:add(transport={"type" =>"TCP", "socket-binding" => "jgroups-tcp"})
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=TCPPING)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=MERGE2)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=FD_SOCK,socket-binding=jgroups-tcp-fd)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=FD)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=VERIFY_SUSPECT)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=BARRIER)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=pbcast.NAKACK)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=UNICAST2)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=pbcast.STABLE)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=pbcast.GMS)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=UFC)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=MFC)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=FRAG2)
    /subsystem=jgroups/stack=true-tcp/:add-protocol(type=RSVP)
    /subsystem=jgroups:write-attribute(name=default-stack,value=true-tcp)
    run-batch
    /subsystem=jgroups/stack=true-tcp/protocol=TCPPING/property=initial_hosts/:add(value="localhost[7600],localhost[7700]")
    /subsystem=jgroups/stack=true-tcp/protocol=TCPPING/property=port_range/:add(value=0)
    /subsystem=jgroups/stack=true-tcp/protocol=TCPPING/property=timeout/:add(value=3000)
    /subsystem=jgroups/stack=true-tcp/protocol=TCPPING/property=num_initial_members/:add(value=3)
    ```
    XML:
    ```
    <subsystem xmlns="urn:jboss:domain:jgroups:1.1" default-stack="true-tcp">
    ...
        <stack name="true-tcp">
            <transport type="TCP" socket-binding="jgroups-tcp"/>
            <protocol type="TCPPING">
                <property name="initial_hosts">
                    localhost[7600],localhost[7700]
                </property>
                <property name="port_range">
                    0
                </property>
                <property name="timeout">
                    3000
                </property>
                <property name="num_initial_members">
                    3
                </property>
            </protocol>
            <protocol type="MERGE2"/>
            <protocol type="FD_SOCK" socket-binding="jgroups-tcp-fd"/>
            <protocol type="FD"/>
            <protocol type="VERIFY_SUSPECT"/>
            <protocol type="BARRIER"/>
            <protocol type="pbcast.NAKACK"/>
            <protocol type="UNICAST2"/>
            <protocol type="pbcast.STABLE"/>
            <protocol type="pbcast.GMS"/>
            <protocol type="UFC"/>
            <protocol type="MFC"/>
            <protocol type="FRAG2"/>
            <protocol type="RSVP"/>
        </stack>

    ```

5. Update httpd.conf on each apache.  Below is the example for ews-1.  Be sure to adjust the port on ews-2 to be 6766. Also note the apache permissions to Directory and Location tags and adjust to your needs.
  
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
6. Start commands. Replace BASE_DIR as needed.
    ```
    BASE_DIR/eap-home/bin/standalone.sh -Djboss.node.name=eap6a -c standalone-ha.xml -Djboss.server.base.dir=BASE_DIR/mc_demo/eap-1 -Djboss.home.dir=BASE_DIR/mc_demo/eap-home -Djboss.socket.binding.port-offset=0 
    ```
    ```
    BASE_DIR/eap-home/bin/standalone.sh -Djboss.node.name=eap6b -c standalone-ha.xml -Djboss.server.base.dir=BASE_DIR/mc_demo/eap-2 -Djboss.home.dir=BASE_DIR/mc_demo/eap-home -Djboss.socket.binding.port-offset=100
    ```

7. Run the Apache servers. 

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