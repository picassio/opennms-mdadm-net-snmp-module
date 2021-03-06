!!! Some blocks are not displaying on github.com. 
!!! Please check this file as raw.
!!! https://raw.github.com/dinde/opennms-mdadm-net-snmp-module/master/README.md

We were looking forward for a decent solution to monitor our software raid (mdadm) server running under Linux (Debian & Centos to be specific).
Many different implementations were avalaible, we wanted to pick one that respect the spirit of opennms avoiding a script to be called on each check.
Snmp polling is realy performant and scalable in Opennms so it mades sense to us to find a solution using snmp.
The other problem is that we wanted an easy to implement solution and we have found this excellent project: https://github.com/stefansaraev/snmp-swraid
This implementation works as a net-snmp module, do not forget to consult README.original for more informations about it.
Basicly, we compiled it once on a debian server with the correct dependencies then copied the .so file on each node with a software raid array running.
Compilation

Dependencies on Debian: libsnmp-dev - Download the source or check it out with git then:

 # tar zxvf snmp-swraid.tar.gz &&  cd snmp-swraid
 # ln -s /usr/include/net-snmp .
 # make

You'll get: swRaidPlugin.so
Server to monitor with mdadm/net-snmp

On debian/Centos:

 # cp -ar SWRAID-MIB.txt /usr/share/mibs/netsnmp/
 # cp -ar swRaidPlugin.so /usr/lib/

- Edit /etc/snmp/snmpd.conf file and add:

 dlmod swRaidMIB /usr/lib/swRaidPlugin.so

- Restart snmpd service

Wait a minute then you can interrogate the OID:

 # snmpwalk -v2c -c public your-mdadm.server.tld .1.3.6.1.4.1.2021.13.18

The output should be like this:

iso.3.6.1.4.1.2021.13.18.1.1.1.1 = INTEGER: 1
iso.3.6.1.4.1.2021.13.18.1.1.1.2 = INTEGER: 2
iso.3.6.1.4.1.2021.13.18.1.1.2.1 = STRING: "md1"
iso.3.6.1.4.1.2021.13.18.1.1.2.2 = STRING: "md0"
iso.3.6.1.4.1.2021.13.18.1.1.3.1 = STRING: "raid1"
iso.3.6.1.4.1.2021.13.18.1.1.3.2 = STRING: "raid1"
iso.3.6.1.4.1.2021.13.18.1.1.4.1 = STRING: "sdb3[0] sda3[1]"
iso.3.6.1.4.1.2021.13.18.1.1.4.2 = STRING: "sdb1[0] sda1[1]"
iso.3.6.1.4.1.2021.13.18.1.1.5.1 = INTEGER: 2
iso.3.6.1.4.1.2021.13.18.1.1.5.2 = INTEGER: 2
iso.3.6.1.4.1.2021.13.18.1.1.6.1 = INTEGER: 2
iso.3.6.1.4.1.2021.13.18.1.1.6.2 = INTEGER: 2
iso.3.6.1.4.1.2021.13.18.100.0 = INTEGER: 0
iso.3.6.1.4.1.2021.13.18.101.0 = ""

Integration with Opennms

If you have more than 4 arrays per node, you'll have to add the definition on poller and capsd configuration, incrementing the OID.

1 / Modify /etc/opennms/capsd-configuration.xml and add:

 <protocol-plugin protocol="Mdadm_Array_1" class-name="org.opennms.netmgt.capsd.plugins.SnmpPlugin" scan="on" user-defined="false">
     <property key="vbname" value=".1.3.6.1.4.1.2021.13.18.1.1.1.1" />
     <property key="timeout" value="3000"/>
     <property key="retry" value="2"/>
 </protocol-plugin>
 <protocol-plugin protocol="Mdadm_Array_2" class-name="org.opennms.netmgt.capsd.plugins.SnmpPlugin" scan="on" user-defined="false">
     <property key="vbname" value=".1.3.6.1.4.1.2021.13.18.1.1.1.2" />
     <property key="timeout" value="3000"/>
     <property key="retry" value="2"/>
 </protocol-plugin>
 <protocol-plugin protocol="Mdadm_Array_3" class-name="org.opennms.netmgt.capsd.plugins.SnmpPlugin" scan="on" user-defined="false">
     <property key="vbname" value=".1.3.6.1.4.1.2021.13.18.1.1.1.3" />
     <property key="timeout" value="3000"/>
     <property key="retry" value="2"/>
 </protocol-plugin>
 <protocol-plugin protocol="Mdadm_Array_4" class-name="org.opennms.netmgt.capsd.plugins.SnmpPlugin" scan="on" user-defined="false">
     <property key="vbname" value=".1.3.6.1.4.1.2021.13.18.1.1.1.4" />
     <property key="timeout" value="3000"/>
     <property key="retry" value="2"/>
 </protocol-plugin>

2 / Modify /etc/opennms/poller-configuration.xml and add:

       <service name="Mdadm_Array_1" interval="300000" user-defined="false" status="on">
           <parameter key="retry" value="1"/>
           <parameter key="timeout" value="3000"/>
           <parameter key="port" value="161"/>
           <parameter key="oid" value=".1.3.6.1.4.1.2021.13.18.1.1.6.1"/>
           <parameter key="operator" value="<="/>
           <parameter key="operand" value="2"/>
       </service>
       <service name="Mdadm_Array_2" interval="300000" user-defined="false" status="on">
           <parameter key="retry" value="1"/>
           <parameter key="timeout" value="3000"/>
           <parameter key="port" value="161"/>
           <parameter key="oid" value=".1.3.6.1.4.1.2021.13.18.1.1.6.2"/>
           <parameter key="operator" value="<="/>
           <parameter key="operand" value="2"/>
       </service>
       <service name="Mdadm_Array_3" interval="300000" user-defined="false" status="on">
           <parameter key="retry" value="1"/>
           <parameter key="timeout" value="3000"/>
           <parameter key="port" value="161"/>
           <parameter key="oid" value=".1.3.6.1.4.1.2021.13.18.1.1.6.3"/>
           <parameter key="operator" value="<="/>
           <parameter key="operand" value="2"/>
       </service>
       <service name="Mdadm_Array_4" interval="300000" user-defined="false" status="on">
           <parameter key="retry" value="1"/>
           <parameter key="timeout" value="3000"/>
           <parameter key="port" value="161"/>
           <parameter key="oid" value=".1.3.6.1.4.1.2021.13.18.1.1.6.4"/>
           <parameter key="operator" value="<="/>
           <parameter key="operand" value="2"/>
       </service>

   <monitor service="Mdadm_Array_1" class-name="org.opennms.netmgt.poller.monitors.SnmpMonitor"/>
   <monitor service="Mdadm_Array_2" class-name="org.opennms.netmgt.poller.monitors.SnmpMonitor"/>
   <monitor service="Mdadm_Array_3" class-name="org.opennms.netmgt.poller.monitors.SnmpMonitor"/>
   <monitor service="Mdadm_Array_4" class-name="org.opennms.netmgt.poller.monitors.SnmpMonitor"/>

6 / Restart opennms and rescan the node. 
