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

[Reserve the NSO Reservable Sandbox](https://devnetsandbox.cisco.com/RM/Diagram/Index/43964e62-a13c-4929-bde7-a2f68ad6b27c?diagramType=Topology)

If you need to revisit some of the NSO Basics, you can [start here](https://developer.cisco.com/learning/lab/learn-nso-the-easy-way/step/1). 

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
packages reload
conf
radius_server_example first-service-instance device dist-rtr01 radius-servers PRIMARY-SERVER acct-port 1000 auth-port 2000 server-ip 172.10.10.10
exit
radius-servers SECONDARY-SERVER acct-port 1000 auth-port 2000 server-ip 10.150.10.10
commit dry-run outformat native
top
radius_server_example second-service-instance device dist-rtr02 radius-servers OTHER-PRIMARY acct-port 1000 auth-port 2000 server-ip 172.101.1.1
exit
radius-servers OTHER-SECONDARY acct-port 1000 auth-port 2000 server-ip 192.10.20.11
top
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
developer@ncs(config)# radius_server_example ?
% No entries found
Possible completions:
  <service-instance:string>
developer@ncs(config)# radius_server_example first-service-instance device dist-rtr01 ?
Possible completions:
  radius-servers  <cr>
developer@ncs(config)# radius_server_example first-service-instance device dist-rtr01 radius-servers ?
Possible completions:
  <server-name:string>
developer@ncs(config)# radius_server_example first-service-instance device dist-rtr01 radius-servers PRIMARY-SERVER ?
Possible completions:
  acct-port  auth-port  server-ip  <cr>
developer@ncs(config)# radius_server_example first-service-instance device dist-rtr01 radius-servers PRIMARY-SERVER acct-port 1000 auth-port 2000 server-ip 172.10.10.10
developer@ncs(config-radius-servers-PRIMARY-SERVER)# exit
developer@ncs(config-radius_server_example-first-service-instance)# radius-servers SECONDARY-SERVER acct-port 1000 auth-port 2000 server-ip 10.150.10.10
developer@ncs(config-radius-servers-SECONDARY-SERVER)# top
developer@ncs(config)# radius_server_example ?
This line doesn't have a valid range expression
Possible completions:
  <service-instance:string>  first-service-instance
developer@ncs(config)# radius_server_example second-service-instance device dist-rtr02 radius-servers OTHER-PRIMARY acct-port 1000 auth-port 2000 server-ip 172.101.1.1
developer@ncs(config-radius-servers-OTHER-PRIMARY)# exit
developer@ncs(config-radius_server_example-second-service-instance)# radius-servers OTHER-SECONDARY acct-port 1000 auth-port 2000 server-ip 192.10.20.11
developer@ncs(config-radius-servers-OTHER-SECONDARY)# top
developer@ncs(config)# commit dry-run outformat native
native {
    device {
        name dist-rtr01
        data aaa new-model
             aaa authentication ppp default group radius
             radius server PRIMARY-SERVER
              address ipv4 172.10.10.10 auth-port 2000 acct-port 1000
             !
             radius server SECONDARY-SERVER
              address ipv4 10.150.10.10 auth-port 2000 acct-port 1000
             !
    }
    device {
        name dist-rtr02
        data aaa new-model
             aaa authentication ppp default group radius
             radius server OTHER-PRIMARY
              address ipv4 172.101.1.1 auth-port 2000 acct-port 1000
             !
             radius server OTHER-SECONDARY
              address ipv4 192.10.20.11 auth-port 2000 acct-port 1000
             !
    }
}
developer@ncs(config)# commit
Commit complete.
developer@ncs(config)#

```



## Service Template Loop Statements

The loop statement example can be found in the NSO Development guide PDF with the installation, in chapter 11 `Templates`. 

Basically loop statements allow us to loop over configuration, without using any Python/Java code. 

The configuration we want to loop over looks like this:
```
radius server SERVERNAME
  address ipv4 22.44.55.66 auth-port 125 acct-port 124
```

And since the IOS NED Yang looks like as follows:
```
    // radius server *
    list server {
      tailf:info "Server configuration";
      tailf:cli-mode-name "config-radius-server";
      key id;
      leaf id {
        type string {
          tailf:info "WORD;;Name for the radius server configuration";
        }
      }

      // radius server * / address
      container address {
        tailf:info "Specify the radius server address";

        // radius server * / address ipv4
        container ipv4 {
          tailf:info "IPv4 Address";
          tailf:cli-compact-syntax;
          tailf:cli-sequence-commands {
            tailf:cli-reset-siblings;
          }
          leaf host {
            tailf:cli-drop-node-name;
            type inet:host {
              tailf:info "Hostname or A.B.C.D;;IPv4 Address of radius server";
            }
          }
          // alias 1-8 aliases for this server (max. 8)
          leaf auth-port {
            tailf:info "UDP port for RADIUS authentication server (default is 1645)";
            tailf:cli-optional-in-sequence;
            type uint16 {
              tailf:info "<0-65535>;;Port number";
              range "0..65535";
            }
          }
          leaf acct-port {
            tailf:info "UDP port for RADIUS accounting server (default is 1646)";
            type uint16 {
              tailf:info "<0-65535>;;Port number";
              range "0..65535";
            }
          }
        }
      }
```

We will need to have a unique key to loop over the server `id`. 



In this service package we have the following data model:
```javascript
module radius_server_example {
  namespace "http://com/example/radius_server_example";
  prefix radius_server_example;

  import ietf-inet-types {
    prefix inet;
  }
  import tailf-ncs {
    prefix ncs;
  }

  list radius_server_example {

    uses ncs:service-data;
    ncs:servicepoint "radius_server_example";

    key service-instance;

    leaf service-instance {
      type string;
    }


    // may replace this with other ways of refering to the devices.
    leaf-list device {
      type leafref {
        path "/ncs:devices/ncs:device/ncs:name";
      }
    }

    list radius-servers{
      key server-name;

      leaf server-name{
        type string;
      }

      leaf server-ip{
      type inet:ipv4-address;
      }

          leaf auth-port {
      type inet:port-number;
    }

    leaf acct-port {
      type inet:port-number;
    }
    }
  }
}
```

Where we have each service instance defined by a unique string, and the server name is the unique key we will loop over. We have a static part of the configuration at the top of our template that is not looped, and a dynamic part which we want to expand out if there is more than radius server:


```xml
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
          <?foreach {/radius-servers}?>
          <server>
            <id>{server-name}</id>
            <address>
              <ipv4>
                <host>{server-ip}</host>
                <auth-port>{auth-port}</auth-port>
                <acct-port>{acct-port}</acct-port>
              </ipv4>
            </address>
          </server>
          <?end?>
        </radius>
      </config>
    </device>
  </devices>
```

