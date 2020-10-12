---
id: 0016
title: Add Instance Object Definition
status: ideation
authors: Manny <mmendez@equinix.com>
---

## Summary

The current hardware data model does not support any OS installation details.
Installing an OS is one of the main focus areas of the Tinkerbell project at the moment.
This exists in the legacy Packet Platform under "instance" in the hardware object and a similar concept is in [metal3][metal3].

This would be used by Boots and Hegel (at least) to serve their needs.

## Goals and not Goals

We have a goal of keeping the Hardware data relatively static.
We need a locaion to place instance/provision/installation-info that changes from provision to provision.
It is not a goal to stick this info in `elastic_data`.

## Content

This proposal is about adding a storage concept for data that is not as permanent as Hardware, and that can persist across multiple Workloads.
This would mimic the EquinixMetal life-cycle of a machine, provisioned and thus occupied until the customer deletes and the machine is deprovisioned.
This life-cycle is also useful in non EM use cases as it allows a worflow to end and a future workflow to act upon the previous os intallation state.

#### Hegel

Hegel currently serves up EC2 style metadata by transforming the JSON blob in hardware.metadata to the compatible form.
The JSON blob in Hardware.metadata is not specified in order to be flexible to non EquinixMetal operators, yet Hegel's transformation logic is fixed.
So in reality this is just an undocumented JSON blob that is actually very useful to the project overall.

#### Boots

Boots is currently dynamically creating the iPXE script from data in Hardware.
We generally want to keep the Hardware data static.
But we would end up needing to modifying it to add ephemeral info, such as IPs (public/private v4 and public v6 as EquinixMetal does today), hostname, os installer to use...

## APIs

The API consists of defining a minimal and extensible Instance object as a field in Hardware, via Protobufs.
We already have a protobuf file in tree ([packet][packet]) that defines the non-tinkerbell Instance JSON object as Protobuf.
This will be used as the starting point, moving fields that are unnecessary for Tinkerbell/Hegel/Boots into a meta field.
If interest builds we can hoist meta values into the Instance protobuf definition.

```protobuf
syntax = "proto3";

option go_package = "packet";

package github.com.tinkerbell.tink.protos.packet;

message Instance {
  string id = 1;
  string state = 2;
  string hostname = 3;

  message OperatingSystem {
    string distro = 1;
    string version = 2;
    string tag = 3;
  }
  OperatingSystem operating_system = 4;

  message IP {
    string address = 1;
    uint32 cidr = 2;
    string gateway = 3;

    enum Family {
      UNKNOWN = 0;
      V4 = 1;
      V6 = 2;
    }
    Family family = 4;

    enum Flags {
      NONE = 0;
      PUBLIC = 1;
      MANAGEMENT = 2;
    };
    uint32 flags = 5;  // bitwise-or of Flags
  }
  repeated IP ips = 5;

  string userdata = 6;
  string crypted_root_password = 7;

  repeated string tags = 8;
  repeated string ssh_keys = 9;
}
```

### Questions

1. Should we stay with bitflags in Instance.OperatingSystem.IP.flags or break them out to bool `public` and `management` ... like currently done?
   I don't particularly like "exploding" flags into fields, but that could just be my once-upon-a-time embedded developer thinking leaking.

2. How to handle extensions?
   Right now I can think of a couple of extensions:
      * OperatingSystem:
        * `AlwaysPXE` for `custom_ipxe` OS
	* any licensing fields
      * Spot Market related fields
      * IQN field
   And the 2 possible ways to handled extensions:
    1. A `bytes` field for embedded messages?
    2. Use `Any`?

[metal3]: https://github.com/metal3-io/baremetal-operator/blob/master/docs/api.md#provisioning
[packet]: https://github.com/tinkerbell/tink/tree/f5cdb83338d6961fb7c4c940918892b639126d0a/protos/packet
