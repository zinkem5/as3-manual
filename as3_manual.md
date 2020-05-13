# The AS3 Manual

![AS3 Logo](./AS3_Robot.png)

----------

Videos
- YouTube: [AS3 Overview](https://youtu.be/cMl3AOtMcUo)
- YouTube: [Using AS3](https://youtu.be/NJjcUUtjnJU)

References
- [API reference](#as3-rest-api-endpoints)
- [AS3 Schema Reference](#as3-schema-reference)

----------

## Introduction
[introduction]: introduction

The goal of this manual is to provide a comprehensive understanding of AS3 from
the ground up.
The guide is broken up into the following sections.

* [Configuring BIG-IP with AS3](#configuring-big-ip-with-as3)

  The first section provides a primer on AS3 and common BIG-IP configurations.

* [Templating AS3 with FAST](#templating-as3-with-fast)  

  The second section will cover some templating strategies, and using F5's FAST templating framework to manage appications within AS3 declarations.

* [Declarative APIs](#declarative-apis)

  A treatise on Declarative APIs and AS3's API philosophy.

* [Reference](#reference)

  This section has a glossary, links, and AS3 API reference.

## Configuring BIG-IP with AS3
[configuring-big-ip-with-as3]: configuring-big-ip-with-as3

### Install AS3

Prerequisites:

* BIG-IP version >=v12.1
* [Latest AS3 RPM release](https://github.com/F5Networks/f5-appsvcs-extension/releases)

AS3 can be installed on BIG-IP like any other iControl LX Extension, from the GUI in the Package Management LX page.

[Complete guide to installing/uninstalling](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html)

### POST to /mgmt/shared/appsvcs/declare

Using a REST client such as Postman, the following declaration can be deployed in an HTTP POST body. This is a simple but functional example of an AS3 declaration to load balance traffic to a pool.

More examples can be found in [AS3 Example Declarations](#as3-example-declarations).

```
POST https://bigip.example.com/mgmt/shared/appsvcs/declare
Authorization: BASIC
Content-Type: application/json

{
  "class": "AS3",
  "action": "deploy",
  "persist": true,
  "declaration": {
    "class": "ADC",
    "schemaVersion": "3.0.0",
    "id": "0123-4567-8910",
    "MyTenant": {
      "class": "Tenant",
      "MyApplication": {
        "class": "Application",
        "template": "http",
        "serviceMain": {
          "class": "Service_HTTP",
          "virtualAddresses": ["10.0.0.1"],
          "pool": "web_pool",
        },
        "web_pool": {
          "class": "Pool",
          "members": [
            {
              "servicePort": 80,
              "serverAddresses": [
                "10.0.1.1",
                "10.0.1.2"
              ]
            }
          ]
        }
      }
    }
  }
}

```

Once the declaration is POSTed, a GET to the same endpoint will retrieve the current declaration. The current declaration is the *source of truth* of the BIG-IP configuration.

### GET /mgmt/shared/appsvcs/declare

```
GET https://bigip.example.com/mgmt/shared/appsvcs/declare
Authorization: BASIC
Content-Type: application/json
```











### Explore AS3, become an expert!

The following links are recommended reading from the official AS3 documentation.


| Link | Description |
| ---- | ---- |
| [AS3 Overview](https://youtu.be/cMl3AOtMcUo) | |
| [Install/Uninstall AS3](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html) | |
| [Authentication and Authorization](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/authentication.html) | |
| [Composing an AS3 Declaration](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/composing-a-declaration.html) | |
| [Validating a Declaration](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/validate.html). | |
| [AS3 Declaration Purpose and Function](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/declaration-purpose-function.html) | |
| [F5 JSON Schema](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/understanding-the-json-schema.html) | |
| [AS3 Schema Reference](#as3-schema-reference) | |
| |
| [Official AS3 Documentation from F5](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/) | Top level documentation link for AS3. |
| [GitHub](https://github.com/F5Networks/f5-appsvcs-extension) | New releases, updated schema, and file issue. |

----------

# Guide

## Preface: Why AS3?

Application Services 3 (AS3) is a declarative JSON API for configuring F5 BIG-IP. AS3 provides a consistent, stable, and transparent way to change your BIG-IP configuration in a single API call.

AS3 is _declarative_, meaning instead of giving it a sequence of instructions, you describe the desired state of your config and AS3 makes it happen. No more worrying about configuration dependencies within the BIG-IP.

AS3 is _idempotent_, meaning the same declaration produces the same results _every time_. The previous configuration has no bearing on the meaning of the current configuration being posted.

Most of all AS3 will remain _backward compatible_ with old declarations. This AS3 team is committed to providing updates that will not break existing declarations.

```
/**
 *  Interesting to note information in this document will
 *  be provided in the form of comments like this one.
 */
```

## Chapter 0: Verifying Installation

[Install AS3](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html)

Using the the BIG-IP's `admin` username and password, the installation can be verified by confirming a 200 status code from the following HTTP call:

```
GET https://bigip.example.com/mgmt/shared/appsvcs/info
Authorization: BASIC
Content-Type: application/json
```

[Authentication and Authorization](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/authentication.html) for more information about AS3's authorization mechanisms.

Now that we have AS3 installed on a BIG-IP, we can start working with the BIG-IP configuration. In the next chapter, we'll learn about AS3's primary API endpoint, and how to use it.

## Chapter 1: Understanding AS3's Declarative Model

Before getting into the specifics of writing AS3 declarations, lets briefly take a look at how AS3 works and the underlying configuration model.

At the highest level, AS3 provides an HTTP interface that a _declaration_ can be posted to. The declaration itself contains the desired configuration state of the target BIG-IP. The desired configuration is then applied atomically and success or error is reported to the end user.

In simple terms, the declaration is an 'all or nothing' proposition. We apply all of the changes, or we apply none of the changes, and the declaration itself contains the entire configuration. Once applied to a BIG-IP, the declaration is the _source of truth_ of configuration of that device.

However, with AS3 the declaration does not need to contain the everything on the BIG-IP. It may be broken up into _Tenants_. Each tenant is a complete unit of configuration, everything listed in the tenant config will get applied when that tenant is configured and everything not in that tenant will be removed from the tenant's namespace.

AS3's default update mode is `selective` mode, which means it will only apply configuration changes to the tenants that are specified in the declaration. If tenants Tom, Dave, and Joe exist on the BIG-IP, and I only post Tom and Dave, only Tom and Dave's configuration will be updated.

Tenants can be named however the user likes, but there is has a special tenant called `Common` that can be referenced by all the other tenants. Otherwise, the configuration contained within a tenant is isolated and cannot be shared.

Each tenant is broken down into _Applications_ which can be thought of as a dictionary of every application owned by that particular tenant. Each Application contains the configuration necessary to connect a Virtual IP on BIG-IP to the server nodes in your infrastructure, and any associated policies and other configuration objects. Application names can be anything you want.

Roughly outlined, the entire structure looks like this:

```JSON
{
  "sales": {
    "checkoutApp":{ ... }
  },
  "billing": {
    "generateInvoiceApp":{ ... },
    "deliverInvoiceApp":{ ... }
  }
}
```

The example here shows tenants for 'sales' and 'billing', with respective apps. Inside each application object is the configuration for a complete application.

---

The AS3 declaration must follow syntactic rules, and those rules are formally specified in a document called the _AS3 Schema_, sometimes just referred to as _the schema_. The AS3 Schema describes what is and isn't a valid AS3 declaration, and is used by AS3 as a first step in verifying a valid configuration has been sent. The AS3 Schema defines all the object types used in a declaration. We were introduced to a few of those objects in the last chapter, `ADC`, `Tenant`, `Application`, `Service_HTTP`. Properties inside these classes have special meanings and are used to configure specific components of an application on a BIG-IP. A complete reference can be found in the documentation:

[AS3 Schema Reference](#as3-schema-reference)

AS3 uses a diff engine to determine which configuration changes need to be made to get the configuration from its current state into the state described in the declaration. When a declaration is submitted, AS3 takes a snapshot of the current BIG-IP configuration via iControl REST and converts it to an AS3 declaration.

The declarations are then diffed and AS3 generates a set of TMSH commands that represent the diff. These commands configure declaration components that do not already exist on BIG-IP, and remove those components that do not exist in the declaration. This means that any configuration applied in the BIG-IP GUI will not be preserved if it is not included in subsequent declarations. For this reason, is it not recommended to manually configure any partitions on the BIG-IP which are managed by AS3.


Since AS3 will apply the changes necessary to get the BIG-IP from its current state into the desired state.

```
/**
 *  Because AS3 will make any changes necessary to finish in the state described
 *  in the declaration, manual configuration changes made to BIG-IP will likely
 *  be lost when a new declaration is sent. It is not recommended to make manual
 *  changes to a BIG-IP managed by AS3 except under special circumstances.
 */
```

Except for Common, all AS3 tenant configs are applied serially. AS3 will query the state of an entire tenant, apply the configuration, and move on to the next tenant. Common is an exception, due to shared ownership of objects Common is configured both before and after the list of user tenants.

This has performance implications if you are thinking about ways to organize your applications. Tenants with many applications will take longer to configure, and configuring multiple tenants simultaneously is also slower than configuring only a single tenant. There is a balancing act that must be played based on your use case. If the user commonly configures many applications that are organized into the same group, they may want to use a tenant for these applications. If the user often applies small changes single applications that have no logical grouping, a tenant per application model can work best.

---

Sometimes, you can point to other places in the declaration when specifying an object. This can be useful for sharing objects between configuration, or keeping your declaration maintainable and readable. We saw an example in the last chapter. The Application declaration was as follows:

```json
"MyApplication": {
    "class": "Application",
    "template": "http",
    "serviceMain": {
        "class": "Service_HTTP",
        "virtualAddresses": [
            "10.0.0.1"
        ],
        "pool": "web_pool"
    },
    "web_pool": {
        "class": "Pool",
        "members": [
            {
                "servicePort": 80,
                "serverAddresses": [
                    "10.0.1.1",
                    "10.0.1.2"
                ]
            }
        ]
    }
}
```

This is an example of the simplest reference in an AS3 declaration. Inside the `Application` object is a property called `web_pool`, and the `pool` property of `serviceMain` has a value of `web_pool` pointing to the value specified inside the Application. These properties are only available to the application they are defined within. This pool could be used for another vip inside this application, but could not be used by other application objects.

For more details, and to see other ways to 'point' to objects on the BIG-IP and in the declaration, the following documentation is recommended:

[AS3 Declaration Purpose and Function](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/declaration-purpose-function.html)

[F5 JSON Schema](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/understanding-the-json-schema.html)

## Chapter 2: Declaration Authoring Basics

### AS3 /declare endpoint

This section focuses on basic interactions with AS3's primary HTTP endpoint `/mgmt/shared/appsvcs/declare` and using this endpoint to apply a simple BIG-IP configuration. This endpoint supports two primary methods: GET and POST. A GET request will return a JSON object with the declaration for the current BIG-IP configuation. Let's try that now.

```
GET https://bigip.example.com/mgmt/shared/appsvcs/info
Authorization: BASIC
Content-Type: application/json
```

If this is the first time using AS3, on an unconfigured BIG-IP, this endpoint will return a `204` status code with no body. This means AS3 is installed, and there is no configuration present on BIG-IP. If BIG-IP has previously been configured through AS3, the response body will contain a JSON object with the current BIG-IP configuration, also known as a declaration.

This endpoint also accepts POST requests, with a declaration specified in the body. Let's take a look at the elements of that declaration, and how to build one from scratch.

### AS3 Declarations (from scratch)

An AS3 declaration is a JSON object that conforms to a JSON schema, and can be edited in any text editor, or directly in Postman. We publish the schema and it can be used with VS Code in particular to provide hints while authoring. Instructions to use this feature are outlined in [Validating a Declaration](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/validate.html).

With text editors at hand... lets take a look at our first AS3 declaration.


```json
{
  "class":"ADC",
  "schemaVersion": "3.0.0"
}
```

This is the most basic declaration, specifying a no-op. AS3 objects often have a `class` property that identifies what type of JSON object we are creating or reading. Each class may have its own set of required and default properties and represents a configuration object in AS3.

This example shows an `ADC` object. The `ADC` object is the root object of an AS3 declaration.

POSTing this declaration to `/mgmt/shared/appsvcs/declare` using Basic authentication will produce a "no change" message from AS3.

```json
{
    "results": [
        {
            "message": "no change",
            "host": "localhost",
            "code": 200
        }
    ],
    "declaration": {
        "class": "ADC",
        "schemaVersion": "3.0.0",
        "id": "autogen_1dea85a4-a485-4ac9-9aef-6c3fd294b65b",
        "updateMode": "selective",
        "controls": {
            "archiveTimestamp": "2019-06-13T07:14:54.667Z"
        }
    }
}
```

```
/**
 * VSCode will provide validation hints if the $schema property is set to the
 * url of the AS3 schema:
 * https://raw.githubusercontent.com/F5Networks/f5-appsvcs-extension/master/schema/latest/as3-schema.json
 */
```

```
/**
 * Some examples in the documentation use an `AS3` object with an `ADC` class
 * object specified at the `declaration` property. The `AS3` class provides some
 * extra features and controls which we will explore later, but for now we can
 * think of the `ADC` class as being the root of our declaration.  
 */
```


Lets actually add some BIG-IP configuration. Here we'll create a simple HTTP virtual service load balancing to a pool with 2 members.

AS3 organizes components inside of Applications, and Applications are grouped together into Tenants. A Tenant might represent a business unit, or a user, or some other grouping of applications owned and maintained together. An Application contains the component configuration connecting your VIP or VIPs to the servers in a pool.

Tenants are described in the declaration by adding a new property to our previous JSON object, and assigning another object to it with the "class" property set to Tenant. Adding a tenant named DavidTennant would look like this:

```json
{
  "class":"ADC",
  "schemaVersion": "3.0.0",
  "DavidTennant": {
  	"class": "Tenant"
  }
}
```

POSTing this declaration again results in a no-change, we've specified a tenant. The active reader will notice this response body is slightly different, the object inside the results array has a new property-- `DavidTennant`. Multiple tenants could be added to this declaration, and each tenant posted will have a corresponding entry in the results array. The tenant property will hold the name of the tenant matching the results object.

We've added a tenant but we still haven't _actually_ configured anything. So lets take a look at our first simple declaration. This declaration describes a basic http server, the bare minimum required to set up a simple load balancing scenario, along with the necessary HTTP information to post.

```
POST https://bigip.example.com/mgmt/shared/appsvcs/declare
Authorization: BASIC
Content-Type: application/json
```
```json
{
  "class": "ADC",
  "schemaVersion": "3.0.0",
  "MyTenant": {
    "class": "Tenant",

    "MyApplication": {
      "class": "Application",
      "template": "http",

      "serviceMain": {
        "class": "Service_HTTP",
        "virtualAddresses": ["10.0.0.1"],
        "pool": "web_pool"
      },

      "web_pool": {
        "class": "Pool",
        "members": [
          {
            "servicePort": 80,
            "serverAddresses": [
              "10.0.1.1",
              "10.0.1.2"
            ]
          }
        ]
      }
    }
  }
}

```

A bunch of stuff got added, lets break it down and take a look at each component.

First off, we see a tenant named "MyTenant". Inside the tenant is a new property "MyApplication" with an object attached, with its own class property "Application". This creates an application called MyApplication inside the tenant MyTenant. Tenant and Application names are totally up to the declaration author. The only constraint is they must start with a letter, and can only contain letters and numbers.

Next we see a new property `"template" : "http"`. This property applies some constraints to the object defined at `serviceMain`, which contains the virtual IP configuration. This template enforces that `serviceMain` must be of class `Service_HTTP` and will provide some default parameters. In this case, the port on the virtual server is set to 80. Other templates include `https`, `tcp`, `udp` can used with serviceMain create to an appropriate service definition. This is a bit of configuration boilerplate, and it's not necessary to understand it completely at this phase, but it is a convention used in many examples.

```
/**
 * By using "template":"generic", a VIP can be given a custom name. Service
 * classes may still be used, but it is up to the user to make sure all
 * properties are appropriately filled in.
 */
```

We know now that `serviceMain` has our service definition, which specifies our VIP, its pools, and its policies. Let's take a closer look at its properties. It is a service, denoted by the class `Service_HTTP`. The virtual addresses to use are listed in the `virtualAddresses` array, and as previously mentioned it will host on port 80 because of the defined template. There is another property specified here called `pool`, as you may have guessed this property specifies the pool the virtual server will load balance to, in this case `web_pool` as defined in the declaration.

In this example, `web_pool` is defined as an Application property. For simple pointers like this, only the property name needs to be specified and AS3 will reference that property in the application for validation and configuration.

Inside `web_pool` we see it is a `Pool` class, specifying `members` with an array of objects with ip/port pairs. This example shows a simple definition where several servers are listening on the same port. The `servicePort` property contains the port, and the `serverAddresses` property specifies a list of IP addresses of the backend servers to load balance to.

## Services

Here is a basic list of service classes available, any of these can be used to specify a Virtual IP load balancing a class of traffic.


`"class":"Service_HTTP"`

A simple HTTP load balancer.

`"class":"Service_HTTPS"`

An HTTPS load balancer, requiring a server side certificate and key.

`"class":"Service_TCP"`

A layer 4 load balancer for TCP connections.

`"class":"Service_UDP"`

A layer 4 load balancer for UDP connections.

`"class":"Service_FastL4"`

A service using F5's Fast L4 feature.


## Pools

Briefly covering a few example scenarios with multiple ip/port combinations on backend servers:

```json
{
  "class": "Pool",
  "members": [
    {
      "servicePort": 80,
      "serverAddresses": [
        "10.0.1.1",
        "10.0.1.2"
      ]
    }
  ]
}
```
This basic example specifies 2 servers each serving 1 port. 10.0.1.1 and 10.0.1.2 both serving HTTP at port 80.


```json
{
  "class": "Pool",
  "members": [
    {
      "servicePort": 8080,
      "serverAddresses": [
        "10.0.1.1",
        "10.0.1.2"
      ]
    },
    {
      "servicePort": 8081,
      "serverAddresses": [
        "10.0.1.1",
        "10.0.1.2"
      ]
    }
  ]
}
```
This specifies 2 servers with 2 ports each. 10.0.1.1 and 10.0.1.2 both serving HTTP at port 8080 and 8081.


```json
{
  "class": "Pool",
  "members": [
    {
      "servicePort": 8080,
      "serverAddresses": [
        "10.0.1.1"
      ]
    },
    {
      "servicePort": 8081,
      "serverAddresses": [
        "10.0.1.2"
      ]
    }
  ]
}
```
This specifies 2 servers with 1 (different) port each-- 10.0.1.1 at 8080 and 10.0.1.2 at 8081.


For more information, please see the AS3 Documentation [Composing an AS3 Declaration](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/composing-a-declaration.html)

### POSTing to AS3

POSTing the declaration to `/mgmt/shared/appsvcs/declare` from the previous section should yield the following response:

```json
{
    "results": [
        {
            "message": "success",
            "lineCount": 24,
            "code": 200,
            "host": "localhost",
            "tenant": "MyTenant",
            "runTime": 716
        }
    ],
    "declaration": {
        "class": "ADC",
        "schemaVersion": "3.0.0",
        "MyTenant": {
            "class": "Tenant",
            "MyApplication": {
                "class": "Application",
                "template": "http",
                "serviceMain": {
                    "class": "Service_HTTP",
                    "virtualAddresses": [
                        "10.0.0.1"
                    ],
                    "pool": "web_pool"
                },
                "web_pool": {
                    "class": "Pool",
                    "members": [
                        {
                            "servicePort": 80,
                            "serverAddresses": [
                                "10.0.1.1",
                                "10.0.1.2"
                            ]
                        }
                    ]
                }
            }
        },
        "id": "autogen_337bcb3c-ba21-48f1-bad4-7ef6d6aae50b",
        "updateMode": "selective",
        "controls": {
            "archiveTimestamp": "2019-06-13T07:40:26.017Z"
        }
    }
}
```

AS3 responds with a results array member corresponding to the tenant that was posted (showing success), as well as the now current declaration. The current declaration can be fetched at any time using the `GET` method

```
GET https://bigip.example.com/mgmt/shared/appsvcs/declare
Authorization: BASIC
Content-Type: application/json
```

In this section all the steps for basic interaction with AS3 have been described. In the following sections, we will take a closer look at how AS3 works and conventions used when authoring declarations.



## Chapter 3: BIG-IP Features in AS3

Here we will use conventions we already know about to introduce some commonly used BIG-IP components to the declaration. At the end of the chapter we will have a declaration that contains many BIG-IP features that can be used as a starting point when writing new declarations.

### Adding https

_In the application:_

Using `"template": "https"` in conjunction with will automatically host the virtual server on port 443. Service_HTTPS provides an HTTP redirect rule on port 80.

2 new additional properties will be added, a TLS_Server class and a Certificate class. In the example below, the TLS_Server is called `webtls` and the certificate `webcert`. The TLS Server is simple, and points to the certificate.

```json
"webtls" : {
  "class": "TLS_Server",
  "certificates": [{
    "certificate": "webcert"
  }]
},
```

The Certificate will also be added:

```json
"webcert": {
   "class": "Certificate",
   "remark": "in practice we recommend using a passphrase",
   "certificate": " {{ User Generated Certificate }}",
   "privateKey": " {{ Generated Private Key matching Certificate }} "
   }
```

Certificates can be protected by passphrase, but this declaration does not use a passphrase protected key.

_In the service:_

Using `"class" : "Service_HTTPS"` will provide the base for the HTTPS server, and take advantage of the https template being used.

A `"serverTLS"` property to the Service points at the TLS_Server, `webtls`.

Putting it all together we have:

```json
{
  "class": "ADC",
  "schemaVersion": "3.0.0",
  "MyTenant": {
    "class": "Tenant",

    "MyApplication": {
      "class": "Application",
      "template": "https",

      "serviceMain": {
        "class": "Service_HTTPS",
        "virtualAddresses": ["10.0.0.1"],
        "pool": "web_pool",

        "serverTLS" : "webtls"

      },

      "web_pool": {
        "class": "Pool",
        "members": [
          {
            "servicePort": 80,
            "serverAddresses": [
              "10.0.1.1",
              "10.0.1.2"
            ]
          }
        ]
      },

      "webtls" : {
        "class": "TLS_Server",
        "certificates": [{
          "certificate": "webcert"
        }]
      },

      "webcert": {
         "class": "Certificate",
         "remark": "in practice we recommend using a passphrase",
         "certificate": " {{ User Generated Certificate }}",
         "privateKey": " {{ Generated Private Key matching Certificate }} "
      }

    }
  }
}

```

### Adding a Monitor to the Pool

Monitors are very easy to add, we simply need to add a new `monitor` property to the Pool. It is an array of monitor types, in this case we will use http.

```
{
  "class": "Pool",
  "monitors": [
    "http"
  ],
```

HTTPS with a Monitor:

```json
{
  "class": "ADC",
  "schemaVersion": "3.0.0",
  "MyTenant": {
    "class": "Tenant",

    "MyApplication": {
      "class": "Application",
      "template": "https",

      "serviceMain": {
        "class": "Service_HTTPS",
        "virtualAddresses": ["10.0.0.1"],
        "pool": "web_pool",
        "serverTLS" : "webtls"
      },
      "web_pool": {
        "class": "Pool",

        "monitors": [
          "https"
        ],

        "members": [
          {
            "servicePort": 80,
            "serverAddresses": [
              "10.0.1.1",
              "10.0.1.2"
            ]
          }
        ]
      },
      "webtls" : {
        "class": "TLS_Server",
        "certificates": [{
          "certificate": "webcert"
        }]
      },
      "webcert": {
         "class": "Certificate",
         "remark": "in practice we recommend using a passphrase",
         "certificate": " {{ User Generated Certificate }}",
         "privateKey": " {{ Generated Private Key matching Certificate }} "
      }

    }
  }
}
```

### Apply WAF Profile

WAF policies can be applied using the `policyWAF` property of the Application class.

At time of writing, WAF Policies must be created through the BIG-IP GUI and exported/imported through traditional means to move them to the BIG-IP assigned to enforce them. AS3 is not yet capable of WAF Policy Composition. This means we need to point to an object on the BIG-IP that AS3 is unaware of.

In this case we can use a `bigip` pointer to a path BIG-IP, `/Common/base_policy.xml`, which holds our desired WAF Policy.

```json
{
  "class": "Service_HTTPS",
  "policyWAF": {
    "bigip": "/Common/base_policy.xml"
  }
}
```

Full example of HTTPS with WAF

```json
{
  "class": "ADC",
  "schemaVersion": "3.0.0",
  "MyTenant": {
    "class": "Tenant",

    "MyApplication": {
      "class": "Application",
      "template": "https",

      "serviceMain": {
        "class": "Service_HTTPS",
        "virtualAddresses": ["10.0.0.1"],
        "pool": "web_pool",
        "serverTLS" : "webtls",

        "policyWAF": {
          "bigip": "/Common/base_policy.xml"
        }

      },
      "web_pool": {
        "class": "Pool",
        "monitors": [
          "https"
        ],
        "members": [
          {
            "servicePort": 80,
            "serverAddresses": [
              "10.0.1.1",
              "10.0.1.2"
            ]
          }
        ]
      },
      "webtls" : {
        "class": "TLS_Server",
        "certificates": [{
          "certificate": "webcert"
        }]
      },
      "webcert": {
         "class": "Certificate",
         "remark": "in practice we recommend using a passphrase",
         "certificate": " {{ User Generated Certificate }}",
         "privateKey": " {{ Generated Private Key matching Certificate }} "
      }

    }
  }
}
```

### Apply iRule

The `iRules` property can be set to an array in the Service definition.

```json
{
  "class":"Application",
  "serviceMain": {
    "class": "Service_HTTPS",
    "pool":"web_pool",
    "iRules": [
      "choose_pool"
    ]
  },
  "web_pool": {
    "class": "Pool",
  },
  "pvt_pool": {
    "class": "Pool",
  },
  "choose_pool": {
    "class": "iRule",
    "iRule": "when CLIENT_ACCEPTED {\nif {[IP::client_addr] starts_with \"10.\"} {\n pool `*pvt_pool`\n }\n}"
  }
}
```

The AS3 iRule definition is the object with `"class":"iRule"`. It has a property `iRule` where any iRule TCL code may be specified.

In this example, the iRule text will route the request to a different pool called `pvt_pool` for requests on the local network, those starting with `10.`. `pvt_pool` is specified in the AS3 declaration, and referenced by the iRule text. If the condition matches, the traffic is routed to `pvt_pool`.

AS3 also provides a method of referencing existing iRules on BIG-IP using a `bigip` pointer as in the WAF Policy example.

### Apply Endpoint Policies
```json
{
  "class": "Application",
  "serviceMain": {
    "class": "Service_HTTP",
    "policyEndpoint": "forward_policy"
  },
  "web_pool": {
    "class": "Pool",
  },
  "forward_policy": {
    "class": "Endpoint_Policy",
    "rules": [{
      "name": "forward_to_pool",
      "conditions": [{
        "type": "httpUri",
        "path": {
          "operand": "contains",
          "values": ["example.com"]
        }
      }],
      "actions": [{
        "type": "forward",
        "event": "request",
        "select": {
          "pool": {
            "use": "web_pool"
          }
        }
      }]
    }]
  }
}
```

iRules provide flexibility, but Endpoint Policies are much faster. In this example, we set up an endpoint policy in AS3 that will forward HTTP requests with an appropriate host header to the specified pool.

Endpoint policies are a little more complex, but can be composed in json. The backbone of this structure looks like this, a list of 'rules', each rule with their own pair of lists, one with conditions and another with actions.

```json
{
  "class":"Endpoint_Policy",
  "rules": [
    {
      "conditions": [],
      "actions": []
    },
    {
      "conditions": [],
      "actions": []
    }
  ]
}
```

The rules are applied in order, top to bottom. When conditions are met, actions are taken.

Our example has one condition:

```json
{
  "type": "httpUri",
  "path": {
    "operand": "contains",
    "values": ["example.com"]
  }
}
```
This looks at the httpUri, and looks for 'example.com'. When the condition is matched, the following action is taken:

```json
{
  "type": "forward",
  "event": "request",
  "select": {
    "pool": {
      "use": "web_pool"
    }
  }
}
```

This action forwards HTTP requests to web_pool.

This Endpoint Policy forwards all traffic with example.com in the URI to the pool, and drops traffic that does not meet the criteria.

## Chapter 4: Service Discovery

AS3 offers tools for automatically adding public cloud resources to BIG-IP LTM Pools. There is a Pool object that configures a poller that will poll AWS, Azure, and Google Cloud for new members. By tagging or labelling resources in the environment, AS3 will automatically add these to pools configured with that tag or label.

The example Pool definition here will work with a BIG-IP VE in AWS to discover EC2 nodes in the same region. The tag is specified with the `tagKey` and `tagValue` properties in the member definition.

```
{
    "awsPool": {
        "class": "Pool",
        "members": [
            {
                "servicePort": 80,
                "addressDiscovery": "aws",
                "region": "us-east-1",
                "updateInterval": 20,
                "tagKey": "color",
                "tagValue": "blue",
                "addressRealm": "public"
            }
        ]
    }
}
```
For more details covering different scenarios, see [Service Discovery in AS3](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/service-discovery.html).





## Templating AS3 with FAST
[templating-as3-with-fast]: templating-as3-with-fast


## Chapter 5: Templating

At scale, configuring BIG-IP with hand written AS3 declarations for every application and service deployed may get tedious. Many will resort to copy pasting a few basic Application objects and tweaking them to fit their needs.

The copy-paste workflow can be streamlined without much effort by using a templating system. Here we take a look at one such templating system developed for AS3. It uses mustache to parse logicless templates, then generates a simple schema describing the template variables. The schema can be used to auto-generate a command line interface or a REST API, as well as HTML forms.

The goal of a template system is to take the complexity and repetition out of creating and modifying declarations. We want to minimize the amount of information the declaration author needs to have to get the desired output declaration.

A template system is composed of two pieces, a template syntax for identifying places in the template text for substitution, and a renderer for producing a valid declaration. The user provides parameters to the renderer and receives a valid AS3 declaration in response.

An example template:

```mustache
{
  "class": "AS3",
  "action": "deploy",
  "persist": true,
  "declaration": {
    "class": "ADC",
    "schemaVersion": "3.0.0",
    "id": "0123-4567-8910",
    "label": "Sample 1",
    "remark": "Basic HTTP with Monitor",
    "{{tenant_name}}": {
      "class": "Tenant",
      "{{application_name}}": {
        "class": "Application",
        "template": "http",
        "serviceMain": {
          "class": "Service_HTTP",
          "virtualAddresses": ["{{virtual_address}}"],
          "pool": "web_pool",
          "enable": {{enable::boolean}}
        },
        "web_pool": {
          "class": "Pool",
          "monitors": [
            "http"
          ],
          "members": [
            {
              "servicePort": 80,
              "serverAddresses": {{server_addresses::array}}
            }
          ]
        }
      }
    }
  }
}
```

Variables are encoded by surrounding them with curly braces, `{{` and `}}`. This declaration could be copied into Postman, and Postman variables work nicely with mustache for exporting. This allows prototyping and testing various AS3 declaration structures in Postman, and copying them into the template system. Contact the author of this document for information on auto-importing postman collections into a templating environment.

In this template some variables are annotated with a type, following the double colon. By default, all variables are treated as strings. It is useful in the schema generation phase to have types specified. This enables the schema to provide richer information for auto-generating front end html forms.

In the example, the template would be loaded into the renderer and when the user invokes it, they need to provide a `tenant_name`, `application_name`, `virtual_address`, ... This is called a 'view'. The view is passed to the renderer, and the renderer outputs a valid AS3 declarations using the values in the view.

Those familiar with AS3 will recognize that to host multiple applications in a single Tenant, we cannot simply POST the rendered declaration to AS3 directly. The application will need to be added to the existing configuration and POSTed back to AS3.

A primary task of the system is to fetch or keep track of AS3 declarations on BIG-IP. When the user wants to deploy a template, the AS3 declaration is recalled from memory or fetched from BIG-IP, the template application is stitched into the existing configuration, and the resulting configuration is applied to BIG-IP.

```
/**
 *  AS3 has an optimistic lock mechanism to prevent side effects of a bad base config. Configuration
 * can be fetched from AS3 with a hash provided using the query parameter showHash=true. This hash can
 * be used when POSTing the modified declaration, and if any changes have been made to the state of AS3
 * while modifying the declaration, the declaration will be rejected. This ensures the client does not
 * accidentally clobber someone else's changes.
 *
 */
```

The reference implementation can be found __here__.

**Using Templates to Manage Configuration Complexity**

One way to manage the variety of configurations in an environment is to always deploy using templates. At an early stage, a network engineer or architect can discuss appropriate configuration patterns to use in their infrastructure. From these patterns, templates can be created that enable a wider audience (such as DevOps) to deploy an application with BIG-IP.



# Declarative APIs
[declarative-apis]: declarative-apis

AS3 is a declarative API.

Idempotence.

Transparency of state.











## Chapter 6: Advanced Features

- references on BIG IP
- URLs that fetch from HTTP endpoints



## Authorization Methods and RBAC

AS3 is available only to users with admin priviledges. There is no way for AS3 to give permissions for a portion of a declaration. It is intended that permissions are handled by an upstream system that is then calling AS3 in it's control plane.

[Authentication and Authorization](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/authentication.html) for more information about AS3's authorization mechanisms.

----------

## Reference
[reference]: reference

* [AS3 Schema Reference](#as3-schema-reference)
* [AS3 Example Declarations](#AS3-Example-Declarations)
* [Glossary](#glossary)
* [Troubleshooting Links](#troubleshooting-links)
* [AS3 REST API Endpoints](#AS3-REST-API-Endpoints)

----------

### AS3 Schema Reference

[AS3 Schema Reference](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/schema-reference.html)

On the other side of this link is a complete reference with every class

----------

### AS3 Example Declarations

[Basic Examples](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/examples.html)

[Intermediate Examples](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/declarations/)

[A Declaration Using All Properties](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/declarations/all-properties.html)

----------

### Troubleshooting Links

[Troubleshooting](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/troubleshooting.html)

[Warnings, Notes, and Tips](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/tips-warnings.html)

[Known Issues](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/known-issues.html)

[File an issue](https://github.com/F5Networks/f5-appsvcs-extension/issues)

----------

### Glossary

**Application** - A logical grouping of BIG-IP component configuration, usually grouped together for a shared function to deliver a particular network application or service. The simplest application is one Virtual Server attached to one Pool.

**AS3** - Application Services 3. An iControl LX Extension that can be installed on BIG-IP to simplify managing a BIG-IP configuration.

**AS3 Class** - An object defined in the AS3 schema, each class has a `class` property with a class name, such as `ADC` or `Application`.

**AS3 Schema** - The master definition of all the properties that can be used in an AS3 declaration to configure a BIG-IP. The AS3 schema represents components that can be configured through AS3.

**BIG-IP** - An Application Delivery Controller built by F5 Networks.

**Declaration** - A JSON object conforming to the AS3 schema containing a specific BIG-IP configuration. The declaration represents an actual BIG-IP configuration.

**iControl REST** (ICR) - An alternative REST interface for interacting with BIG-IP.

**iControl LX Extensions** (icrlx) - RPMs containing Node.js applications that can be installed to extend BIG-IPs base functionality.

**Service Discovery** - A set of features in AS3 for automatically adding new servers to a pool. This can be used in conjunction with AWS, Azure, and other cloud services to automatically detect new pool members as they are created.

**Tenant** - A group of applications, typically grouped based on who owns a set of applications.

----------

### AS3 REST API Endpoints

All endpoints use `/mgmt/shared/appsvcs` as a base.

-------------

### /declare

#### GET
##### Description:

Use GET to retrieve the current declaration or an index of stored declarations. By default, this endpoint returns the base declaration for all Tenants on target localhost.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| show | query | base will return the current declaration with secrets encrypted.<br/> full returns the declaration with all default schema properties populated as it was posted.<br> expanded will evaluate URLs, base64s, and references to their values.  | No | string |
| age | query | If you use ?age=0-15 asks for a declaration of the given age (0 means most-recently deployed), and “list” asks for a list of available declarations with their ages. If this element is not present, the default age is 0. By default, list only shows 4 declarations, this is configurable using historyLimit in the AS3 class./  | No | string |

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | OK |
| 204 | No Content |

-------------

#### POST
##### Description:

POST can be used to apply a declaration to AS3.<br/><br/> Using the AS3 class, this endpoint can be used to deploy a configuration to a target BIG-IP, dry-run a config for validation, or retreive a declaration from a remote AS3 instance. Read about the AS3 class for a more comprehensive explanation of the functions available through POST.<br/><br/> Using the ADC class, a configuration can be applied to the BIG-IP running the target instace of AS3.<br/>


##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| show | query | Same as GET. | No | string |
| async | query | Setting async to true causes AS3 to respond immediately with a 202 status and a task ID. The task can be queried while the configuration is applied in the background. | No | string |

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | OK |
| 207 | Multistatus (check response body) |
| 422 | Unprocessable Entity |

-------------

#### PATCH
##### Description:

PATCH can be used to change a small snippet of an AS3 declaration.

Declarations can be changed on BIG-IP using a PATCH call to `/declare`. The JSON body of the HTTP PATCH contains a directive for manipulating the declaration, according to the syntax in [RFC6902](https://tools.ietf.org/html/rfc6902). AS3 will apply the patch according to the body, and then deploy the resulting declaration.

PATCH is not a way to get better performance from AS3, and in many cases performance will degrade with frequent use.

More details on how to patch can be found in the [PATCH documentation](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/as3-api.html#method-patch).

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | OK |
| 400 | invalid |
| 422 | invalid |

-------------

### /declare/{tenant_list}

#### GET
##### Description:

A comma separated list of tenants can be provided at the end of the path to declare to see specific tenants.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| tenant_list | path | comma separated list of tenants | No | string |
| show | query | Same as GET to declare | No | string |
| age | query | If you use ?age=0-15 asks for a declaration of the given age (0 means most-recently deployed), and “list” asks for a list of available declarations with their ages. If this element is not present, the default age is 0. By default, list only shows 4 declarations, this is configurable using historyLimit in the AS3 class./  | No | string |

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | OK |

#### DELETE
##### Description:

Use DELETE to remove configurations for one or more declared Tenants from the target ADC. If you do not specify any Tenants, DELETE removes all of them, which is to say, it removes the entire declared configuration. Indicate the target device and Tenants to remove by appending elements to the main AS3 URL path (/mgmt/shared/appsvcs/declare). By default (just main URL) DELETE removes all Tenants from target localhost.


##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| tenant_list | path | comma separated list of tenants | No | string |
| show | query | Same as GET to /declare | No | string |

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | OK |

-------------

### /info


#### GET
##### Description:

Contains information about the AS3 version running.

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | OK |

-------------

### /task


#### GET
##### Description:

Used to fetch tasks when using `async=true`

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | OK |

For more information, see the official [AS3 API Doc](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/as3-api.html)
