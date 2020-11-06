# Learn by Doing: Service Template Loops

NSO has a ton of features. This repository is the first of a series which will show a simple use case, along with a feature. The purpose is dual:

1. See NSO applied to a variety of configuration situations and see how versatile it is..
1. Learn something new and have an example to follow

This example is a service package deploying:

**Radius Configuration**

and the feature I am showcasing is:

**Service Template XML Loops**

This example assumes a working knowledge of services and NSO. 

## Installation

[Reserve the NSO Reservable Sandbox](https://blogs.cisco.com/developer/nso-learning-lab-and-sandbox)

Use some type of file transfer program or [VS Code has remote SSH](https://code.visualstudio.com/docs/remote/ssh) (drag and drop the package into the packages directory)

This repo should go here:

`/var/opt/ncs/packages/`

If you are repoducing this elsewhere, here are the version details:
```
using NSO 5.3.0.1
cisco-ios-cli-6.44
on
CSR1000V 16.11.01b
```

After the package is on the preconfigured NSO system install server (10.10.20.49) at the packages folder, reload packages and configure some sample service instances:

```
conf
radius_server_example EMEA device dist-rtr01 acct-port 1000 auth-port 1005 radius_server_list [ 172.70.70.10 172.80.80.10 ]
radius_server_example APJC device dist-rtr02 acct-port 1000 auth-port 1005 radius_server_list [ 172.90.70.10 172.100.80.10 ]
radius_server_example AMER device internet-rtr01 acct-port 671 auth-port 700 radius_server_list [ 172.30.10.12 172.40.20.22 ]
commit dry-run outformat native
commit
```

here is some sample output:

```
[developer@nso src]$ ncs_cli -C
developer@ncs# packages reload

>>> System upgrade is starting.
>>> Sessions in configure mode must exit to operational mode.
>>> No configuration changes can be performed until upgrade has completed.
>>> System upgrade has completed successfully.
reload-result {
    package cisco-asa-cli-6.8
    result true
}
reload-result {
    package cisco-ios-cli-6.44
    result true
}
reload-result {
    package cisco-iosxr-cli-7.20
    result true
}
reload-result {
    package cisco-nx-cli-5.15
    result true
}
reload-result {
    package radius_server_example
    result true
}
reload-result {
    package resource-manager
    result true
}
developer@ncs#
developer@ncs# conf
Entering configuration mode terminal
developer@ncs(config)# radius_server_example EMEA device dist-rtr01 acct-port 1000 auth-port 1005 radius_server_list [ 172.70.70.10 172.80.80.10 ]
developer@ncs(config-radius_server_example-EMEA)# radius_server_example APJC device dist-rtr02 acct-port 1000 auth-port 1005 radius_server_list [ 172.90.70.10 172.100.80.10 ]
developer@ncs(config-radius_server_example-APJC)# radius_server_example AMER device internet-rtr01 acct-port 671 auth-port 700 radius_server_list [ 172.30.10.12 172.40.20.22 ]
developer@ncs(config-radius_server_example-AMER)# commit dry-run outformat native
native {
    device {
        name dist-rtr01
        data aaa new-model
             aaa authentication ppp default group radius
             radius server EMEA
              address ipv4 172.70.70.10 auth-port 1005 acct-port 1000
              address ipv4 172.80.80.10 auth-port 1005 acct-port 1000
             !
    }
    device {
        name dist-rtr02
        data aaa new-model
             aaa authentication ppp default group radius
             radius server APJC
              address ipv4 172.90.70.10 auth-port 1005 acct-port 1000
              address ipv4 172.100.80.10 auth-port 1005 acct-port 1000
             !
    }
    device {
        name internet-rtr01
        data aaa new-model
             aaa authentication ppp default group radius
             radius server AMER
              address ipv4 172.30.10.12 auth-port 700 acct-port 671
              address ipv4 172.40.20.22 auth-port 700 acct-port 671
             !
    }
}
developer@ncs(config-radius_server_example-AMER)# commit
Commit complete.
developer@ncs(config-radius_server_example-AMER)#

```



## Service Template Loop Statements

The loop statement example can be found in the NSO Development guide PDF with the installation, in chapter 11 `Templates`. 

Basically loop statements allow us to loop over configuration, without using any Python/Java code. 

In this service package we have the following data model:
```javascript
  list radius_server_example {

    uses ncs:service-data;
    ncs:servicepoint "radius_server_example";

    key region;

    leaf region {
      type enumeration {
        enum "AMER";
        enum "APJC";
        enum "EMEA";
      }
    }


    // may replace this with other ways of refering to the devices.
    leaf-list device {
      type leafref {
        path "/ncs:devices/ncs:device/ncs:name";
      }
    }

    leaf-list radius_server_list{
      type inet:ipv4-address;
    }
 


    leaf auth-port {
      type inet:port-number;
    }
  
    leaf acct-port {
      type inet:port-number;
    }
  
  }

```

Where we have each service instance defined by what region the server is in. This unique key will be the server ID in the configuration template. 

There may be one or many radius server IP addresses, so we use a leaf-list. If there is more than one server, the statement `<?foreach {/radius_server_list}?>` loops for each IP in the leaf list input, with an `<?end?>` at the end of that block of XML. 


```xml
<config>
<aaa xmlns="urn:ios">
  <new-model/>
  <authentication>
    <ppp>
      <name>default</name>
      <group>radius</group>
    </ppp>
  </authentication>
</aaa>

<radius xmlns="urn:ios">
    <server>
      <id>{/region}</id>
    <?foreach {/radius_server_list}?>
      <address>
        <ipv4>
          <host>{current()}</host>
          <auth-port>{/auth-port}</auth-port>
          <acct-port>{/acct-port}</acct-port>
        </ipv4>
      </address>
    <?end?>
    </server>
  </radius>
</config>
```

We are also using the `current()` XPATH function to grab the IP address of whatever the current loop iteration IP address is and plug that into the `host` input leaf. So the resulting configuration looks like this:

```
  aaa new-model
  aaa authentication ppp default group radius
  radius server EMEA
  address ipv4 172.70.70.10 auth-port 1005 acct-port 1000
  address ipv4 172.80.80.10 auth-port 1005 acct-port 1000
```