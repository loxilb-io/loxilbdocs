# License 
This document is based on the original work by [GOBGP](https://github.com/osrg/gobgp.git).
Changes have been made to the original document.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.


# Policy Configuration

This page explains LoxiLB with GoBGP policy feature for controlling the route
advertisement. It might be called Route Map in other BGP
implementations.

And This document was written with reference to [this goBGP offitial document.](https://github.com/osrg/gobgp/blob/master/docs/sources/policy.md)

We explain the overview firstly, then the details.

## Prerequisites
Assumed that you run loxilb with `-b` option. Or If you control loxilb through kube-loxilb, be sure to set the `--set-bgp` option in the kube-loxilb.yaml file.

```bash
docker run -u root --cap-add SYS_ADMIN --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest -b
```
or in the kube-loxilb.yaml

```
        args:
            - --loxiURL=http://12.12.12.1:11111
            - --externalCIDR=123.123.123.1/24
            - --setBGP=65100

```
(Note) Currently, gobgp does not support the Policy command in global state. Therefore, only the policy for neighbors is applied, and we plan to apply the global policy through additional development.
To apply a policy in a neighbor, you must form a peer by adding the `route-server-client` option when using gobgp in loxilb. This does not provide a separate API and will be provided in the future.
For examples in gobgp, please refer to the following [documents](https://github.com/osrg/gobgp/blob/master/docs/sources/route-server.md).

## Contents

- [Policy Configuration](#policy-configuration)
  - [Prerequisites](#prerequisites)
  - [Contents](#contents)
  - [Overview](#overview)
  - [Route Server Policy Model](#route-server-policy-model)
  - [Policy Structure](#policy-structure)
  - [Configure Policies](#configure-policies)
    - [1. Defining defined-sets](#1-defining-defined-sets)
      - [prefix-sets](#prefix-sets)
        - [Examples](#examples)
      - [neighbor-sets](#neighbor-sets)
        - [Examples](#examples-1)
    - [2. Defining bgp-defined-sets](#2-defining-bgp-defined-sets)
      - [community-sets](#community-sets)
        - [Examples](#examples-2)
      - [ext-community-sets](#ext-community-sets)
        - [Examples](#examples-3)
      - [as-path-sets](#as-path-sets)
        - [Examples](#examples-4)
    - [3. Defining policy-definitions](#3-defining-policy-definitions)
      - [Execution condition of Action](#execution-condition-of-action)
        - [Examples](#examples-5)
    - [4. Attaching policy](#4-attaching-policy)
      - [4.1. Attach policy to route-server-client](#41-attach-policy-to-route-server-client)
  - [Policy and Soft Reset](#policy-and-soft-reset)

## Overview

Policy is a way to control how BGP routes inserted to RIB or advertised to
peers. Policy has two parts, **Condition** and **Action**.
When a policy is configured, **Action** is applied to routes which meet
**Condition** before routes proceed to next step.

GoBGP supports **Condition** like `prefix`, `neighbor`(source/destination of
the route), `aspath` etc.., and **Action** like `accept`, `reject`,
`MED/aspath/community manipulation` etc...

You can configure policy by configuration file, CLI or gRPC API.
Here, we show how to configure policy via configuration file.

## Route Server Policy Model

The following figure shows how policy works in
[route server BGP configuration](route-server.md).

![route server policy model](photos/policy.png)

In route server mode, **Import** and **Export** policies are defined
with respect to a peer.  The **Import** policy defines what routes
will be imported into the master RIB. The **Export** policy defines
what routes will be exported from the master RIB.

You can check each policy by the following commands in the loxilb.

```shell
$ gobgp neighbor <neighbor-addr> policy import
$ gobgp neighbor <neighbor-addr> policy export
```

## Policy Structure

![policy component](photos/policy-component.png)

A policy consists of statements. Each statement has condition(s) and action(s).

Conditions are categorized into attributes below:

- prefix
- neighbor
- aspath
- aspath length
- community
- extended community
- rpki validation result
- route type (internal/external/local)
- large community
- afi-safi in

As showed in the figure above, some of the conditions point to defined sets,
which are a container for each condition item (e.g. prefixes).

Actions are categorized into attributes below:

- accept or reject
- add/replace/remove community or remove all communities
- add/subtract or replace MED value
- set next-hop (specific address/own local address/don't modify)
- set local-pref
- prepend AS number in the AS_PATH attribute

When **ALL** conditions in the statement are `true`, the action(s) in the
statement are executed.

You can check policy configuration by the following commands.

```shell
$ kubectl get bgppolicydefinedsetsservice

$ kubectl get bgppolicydefinitionservice

$ kubectl get bgppolicyapplyservice
```

## Configure Policies

Policy Configuration comes from two parts, definition and attachment. For definition, we have
[defined-sets](#1-defining-defined-sets) and [policy-definition](#3-defining-policy-definitions).
**defined-sets** defines condition item for some of the condition type.
**policy-definitions** defines policies based on actions and conditions.

- **defined-sets**
  A single **defined-sets** entry has prefix match that is named
  **prefix-sets** and neighbor match part that is named **neighbor-sets**. It
  also has **bgp-defined-sets**, a subset of **defined-sets** that defines
  conditions referring to BGP attributes such as aspath. This **defined-sets**
  has a name and it's used to refer to **defined-sets** items from outside.

- **policy-definitions**
  **policy-definitions** is a list of policy. A single element has
  **statements** part that combines conditions with an action.

Below are the steps for policy configuration

1. define defined-sets
    1. define prefix-sets
    2. define neighbor-sets
2. define bgp-defined-sets
    1. define community-sets
    2. define ext-community-sets
    3. define as-path-setList
    4. define large-community-sets
3. define policy-definitions
4. attach neighbor

### 1. Defining defined-sets

defined-sets has prefix information and neighbor information in prefix-sets and
neighbor-sets section, and GoBGP uses these information to evaluate routes.
Defining defined-sets is needed at first.
prefix-sets and neighbor-sets section are prefix match part and neighbor match
part.

- defined-sets example

 ```yaml
 # prefix match part
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-prefix
spec:
  name: "ps1"
  definedType: "prefix"
  prefixList:
    - ipPrefix: "10.33.0.0/16"
      masklengthRange: "21..24"


# neighbor match part
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-neighbor
spec:
  name: "ns1"
  definedType: "neighbor"
  List:
  - "10.0.255.1/32"

 ```

#### prefix-sets

prefix-sets has prefix-set-list, and prefix-set-list has prefix-set-name and
prefix-list as its element. prefix-set-list is used as a condition. Note that
prefix-sets has either v4 or v6 addresses.

**prefix** has 1 element and list of sub-elements.

| Element | Description | Example | Optional |
| --- | --- | --- | --- |
| name | name of prefix-set | "ps1" |  |
| prefixList | list of prefix and range of length |  |  |

**PrefixList** has 2 elements.

| Element | Description | Example | Optional |
| --- | --- | --- | --- |
| ipPrefix | prefix value | "10.33.0.0/16" |  |
| masklengthRange | range of length | "21..24" | Yes |

##### Examples

- example 1
  - Match routes whose high order 2 octets of NLRI is 10.33 and its prefix
    length is between from 21 to 24
  - If you define a prefix-list that doesn't have MasklengthRange, it matches
    routes that have just 10.33.0.0/16 as NLRI.

```yaml
# example 1
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-prefix
spec:
  name: "ps1"
  definedType: "prefix"
  prefixList:
    - ipPrefix: "10.33.0.0/16"
      masklengthRange: "21..24"
      
```

- example 2
  - If you want to evaluate multiple routes with a single prefix-set-list, you
    can do this by adding an another prefix-list like this:
  - This prefix-set-list match checks if a route has 10.33.0.0/21 to 24 or
    10.50.0.0/21 to 24.

```yaml
# example 2
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-prefix
spec:
  name: "ps1"
  definedType: "prefix"
  prefixList:
    - ipPrefix: "10.33.0.0/16"
      masklengthRange: "21..24"
    - ipPrefix: "10.50.0.0/16"
      masklengthRange: "21..24"
```

- example 3
  - prefix-set-name under prefix-set-list is reference to a single prefix-set.
  - If you want to add different prefix-set more, you can add other blocks that
    form the same structure with example 1.

```yaml
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-prefix
spec:
  name: "ps1"
  definedType: "prefix"
  prefixList:
    - ipPrefix: "10.33.0.0/16"
      masklengthRange: "21..24"
---
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-prefix
spec:
  name: "ps2"
  definedType: "prefix"
  prefixList:
    - ipPrefix: "10.50.0.0/16"
      masklengthRange: "21..24"
```

#### neighbor-sets

neighbor-sets has neighbor-set-list, and neighbor-set-list has
neighbor-set-name and neighbor-info-list as its element. It is necessary to
specify a neighbor address in neighbor-info-list. neighbor-set-list is used as
a condition.
*Attention: an empty neighbor-set will match against ANYTHING and not invert based on the match option*

**neighbor** has 1 element and list of sub-elements.

| Element | Description | Example | Optional |
| --- | --- | --- | --- |
| name | name of neighbor | "ns1" |  |
| List | list of neighbor address |  |  |

**neighbor-info-list** has 1 element.

| Element | Description | Example | Optional |
| --- | --- | --- | --- |
| - | neighbor address | "10.0.255.1" |  |

##### Examples

- example 1

```yaml
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-neighbor
spec:
  name: "ns1"
  definedType: "neighbor"
  List:
  - "10.0.255.1/32"
---
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-neighbor
spec:
  name: "ns2"
  definedType: "neighbor"
  List:
  - "10.0.0.0/24"
```

- example 2
  - As with prefix-set-list, neighbor-set-list can have multiple
    neighbor-info-list like this.

```yaml
# example 2
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-neighbor
spec:
  name: "ns1"
  definedType: "neighbor"
  List:
  - "10.0.255.1/32"
  - "10.0.255.2/32"
  ```

- example 3
  - As with prefix-set-list, multiple neighbor-set-lists can be defined.

```yaml
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-neighbor
spec:
  name: "ns1"
  definedType: "neighbor"
  List:
  - "10.0.255.1/32"
---
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-neighbor
spec:
  name: "ns2"
  definedType: "neighbor"
  List:
  - "10.0.254.1/32"
```

### 2. Defining bgp-defined-sets

bgp-defined-sets has Community information, Extended Community
information and AS_PATH information in each Sets section
respectively. And it is a child element of defined-sets.
community-sets, ext-community-sets and as-path-sets section are each match
part. Like prefix-sets and neighbor-sets, each can have multiple sets and each
set can have multiple values.

- bgp-defined-sets example

```yaml
# Community match part
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-community
spec:
  name: "community1"
  definedType: "community"
  List:
  - "65100:10"

# Extended Community match part
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-extcommunity
spec:
  name: "ecommunity1"
  definedType: "extcommunity"
  List:
  - "RT:65100:100"

# AS_PATH match part
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-aspath
spec:
  name: "aspath1"
  definedType: "asPath"
  List:
    - "^65100"
    
# Large Community match part
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-largecommunity
spec:
  name: "lcommunity1"
  definedType: "largecommunity"
  List:
  - "65100:100:100"
```

#### community-sets

community-sets has community-set-name and community-list as its element. The
Community value are used to evaluate communities held by the destination.

| Element | Description | Example | Optional |
| --- | --- | --- | --- |
| name | name of CommunitySet | "community1" |  |
| List | list of community value |  |  |

**community-list** has 1 element.

| Element | Description | Example | Optional |
| --- | --- | --- | --- |
| - | community value | "65100:10" |  |

You can use regular expressions to specify community in community-list.

##### Examples

- example 1
  - Match routes which has "65100:10" as a community value.

```yaml
# example 1
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-community
spec:
  name: "community1"
  definedType: "community"
  List:
  - "65100:10"
```

- example 2
  - Specifying community by regular expression
  - You can use regular expressions based on POSIX 1003.2 regular expressions.

```yaml
# example 2
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-community
spec:
  name: "community2"
  definedType: "community"
  List:
  - "6[0-9]+:[0-9]+"
```

#### ext-community-sets

ext-community-sets has ext-community-set-name and ext-community-list as its
element. The values are used to evaluate extended communities held by the
destination.

| Element | Description | Example | Optional |
| --- | --- | --- | --- |
| name | name of ExtCommunitySet | "ecommunity1" |  |
| List | list of extended community value |  |  |

**List** has 1 element.

| Element | Description | Example | Optional |
| --- | --- | --- | --- |
| - | extended community value | "RT:65001:200" |  |

You can use regular expressions to specify extended community in
ext-community-list. However, the first one element separated by (part of "RT")
does not support to the regular expression. The part of "RT" indicates a
subtype of extended community and subtypes that can be used are as follows:

- RT: mean the route target.
- SoO: mean the site of origin(route origin).
- encap: mean the encapsulation tunnel type, currently gobgp supports the following encap tunnels:
    l2tp3
    gre
    ip-in-ip
    vxlan
    nvgre
    mpls
    mpls-in-gre
    vxlan-gre
    mpls-in-udp
    sr-policy
    geneve
- LB: mean the link-bandwidth (in bytes).

##### Examples

- example 1
  - Match routes which has "RT:65001:200" as a extended community value.

```yaml
# example 1
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-extcommunity
spec:
  name: "ecommunity1"
  definedType: "extcommunity"
  List:
  - "RT:65100:100"
```

- example 2
  - Specifying extended community by regular expression
  - You can use regular expressions that is available in Golang.

```yaml
# example 2
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-extcommunity
spec:
  name: "ecommunity2"
  definedType: "extcommunity"
  List:
  - "RT:6[0-9]+:[0-9]+"
```

#### as-path-sets

as-path-sets has as-path-set-name and as-path-list as its element. The numbers
are used to evaluate AS numbers in the destination's AS_PATH attribute.

| Element | Description | Example | Optional |
| --- | --- | --- | --- |
| name | name of as-path-set | "aspath1" |  |
| List | list of as path value |  |  |

**List** has 1 elements.

| Element | Description | Example | Optional |
| --- | --- | --- | --- |
| - | as path value | "^65100" |  |

The AS path regular expression is compatible with
[Quagga](http://www.nongnu.org/quagga/docs/docs-multi/AS-Path-Regular-Expression.html)
and Cisco. Note Character `_` has special meaning. It is abbreviation for
`(^|[,{}() ]|$)`.

Some examples follow:

- From: `^65100_` means the route is passed from AS 65100 directly.
- Any: `_65100_` means the route comes through AS 65100.
- Origin: `_65100$` means the route is originated by AS 65100.
- Only: `^65100$` means the route is originated by AS 65100 and comes from it
  directly.
- `^65100_65001`
- `65100_[0-9]+_.*$`
- `^6[0-9]_5.*_65.?00$`

##### Examples

- example 1
  - Match routes which come from AS 65100.

```yaml
# example 1
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-aspath
spec:
  name: "aspath1"
  definedType: "asPath"
  List:
    - "^65100"
```

- example 2
  - Match routes which come Origin AS 65100 and use regular expressions to
    other AS.

```yaml
# example 2
apiVersion: bgppolicydefinedsets.loxilb.io/v1
kind: BGPPolicyDefinedSetsService
metadata:
  name: policy-aspath
spec:
  name: "aspath1"
  definedType: "asPath"
  List:
    - "[0-9]+_65[0-9]+_65100$"
```

### 3. Defining policy-definitions

policy-definitions consists of condition and action. Condition part is used to
evaluate routes from neighbors, if matched, action will be applied.

- an example of policy-definitions

```yaml
apiVersion: bgppolicydefinition.loxilb.io/v1
kind: BGPPolicyDefinitionService
metadata:
  name: example-policy
spec:
  name: example-policy
  statements:
  - name: statement1
    conditions:
      matchPrefixSet:
        prefixSet: ps1
        matchSetOptions: any
      matchNeighborSet:
        neighborSet: ns1
        matchSetOptions: invert
      bgpConditions:
        matchCommunitySet:
          communitySet: community1
          matchSetOptions: any
        matchExtCommunitySet:
          communitySet: ecommunity1
          matchSetOptions: any
        matchAsPathSet:
          asPathSet: aspath1
          matchSetOptions: any
        asPathLength:
          operator: eq
          value: 2
        afiSafiIn:
          - l3vpn-ipv4-unicast
          - ipv4-unicast
    actions:
      routeDisposition: accept-route
      bgpActions:
        setMed: "-200"
        setAsPathPrepend:
          as: "65005"
          repeatN: 5
        setCommunity:
          options: add
          setCommunityMethod:
            communitiesList:
              - 65100:20  

```

 The elements of policy-definitions are as follows:

- policy-definitions
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | name | policy's name | "example-policy" |
- statements
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | name | statements's name | "statement1" |
- conditions - match-prefix-set
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | prefixSet | name for defined-sets.prefix-sets.prefix-set-list that is used in this policy | "ps1" |
    | matchSetOptions | option for the check:<br> "any" or "invert". default is "any" | "any" |
- conditions - match-neighbor-set
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | neighborSet | name for defined-sets.neighbor-sets.neighbor-set-list that is used in this policy | "ns1" |
    | matchSetOptions | option for the check:<br> "any" or "invert". default is "any" | "any" |
- conditions - bgp-conditions - match-community-set
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | communitySet | name for defined-sets.bgp-defined-sets.community-sets.CommunitySetList that is used in this policy | "community1" |
    | matchSetOptions | option for the check:<br> "any" or "all" or "invert". default is "any" | "invert" |
- conditions - bgp-conditions - match-ext-community-set
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | communitySet | name for defined-sets.bgp-defined-sets.ext-community-sets that is used in this policy | "ecommunity1" |
    | matchSetOptions | option for the check:<br> "any" or "all" or "invert". default is "any" | "invert" |
- conditions - bgp-conditions - match-as-path-set
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | asPathSet | name for defined-sets.bgp-defined-sets.as-path-sets that is used in this policy | "aspath1" |
    | matchSetOptions | option for the check:<br> "any" or "all" or "invert". default is "any" | "invert" |
- conditions - bgp-conditions - match-as-path-length
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | operator | operator to compare the length of AS number in AS_PATH attribute. <br> "eq","ge","le" can be used. <br> "eq" means that length of AS number is equal to Value element <br> "ge" means that length of AS number is equal or greater than the Value element <br> "le" means that length of AS number is equal or smaller than the Value element | "eq" |
    | value | value used to compare with the length of AS number in AS_PATH attribute | 2 |
- statements - actions
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | routeDisposition | stop following policy/statement evaluation and accept/reject the route:<br> "accept-route" or "reject-route" | "accept-route" |
- statements - actions - bgp-actions
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | setMed | set-med used to change the med value of the route. <br> If only numbers have been specified, replace the med value of route.<br> if number and operater(+ or -) have been specified, adding or subtracting the med value of route. | "-200" |
- statements - actions - bgp-actions - set-community
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | options | operator to manipulate Community attribute in the route | "ADD" |
    | communities | communities used to manipulate the route's community according to options below | "65100:20" |
- statements - actions - bgp-actions - set-as-path-prepend
    
    
    | Element | Description | Example |
    | --- | --- | --- |
    | as | AS number to prepend. You can use "last-as" to prepend the leftmost AS number in the aspath attribute. | "65100" |
    | repeatN | repeat count to prepend AS | 5 |

#### Execution condition of Action

 Action statement is executed when the result of each Condition, including
 match-set-options is all true.
 **match-set-options** is defined how to determine the match result, in the
 condition with multiple evaluation set as follows:

 | Value  | Description                                                               |
 | ------ | ------------------------------------------------------------------------- |
 | any    | match is true if given value matches any member of the defined set        |
 | all    | match is true if given value matches all members of the defined set       |
 | invert | match is true if given value does not match any member of the defined set |

##### Examples

- example 1
  - This policy definition has prefix-set *ps1* and neighbor-set *ns1* as its
    condition and routes matches the condition is rejected.

```yaml
# example 1
apiVersion: bgppolicydefinition.loxilb.io/v1
kind: BGPPolicyDefinitionService
metadata:
  name: policy1
spec:
  name: policy1
  statements:
  - name: statement1
    conditions:
      matchPrefixSet:
        prefixSet: ps1
        matchSetOptions: any
      matchNeighborSet:
        neighborSet: ns1
        matchSetOptions: any
    actions:
      routeDisposition: reject-route
```

- example 2
  - policy-definition has two statements
  - If a route matches the condition inside the first statement(1), GoBGP
    applies its action and quits the policy evaluation.

```yaml
# example 2
apiVersion: bgppolicydefinition.loxilb.io/v1
kind: BGPPolicyDefinitionService
metadata:
  name: policy1
spec:
  name: policy1
  statements:
  - name: statement1
    conditions:
      matchPrefixSet:
        prefixSet: ps1
        matchSetOptions: any
      matchNeighborSet:
        neighborSet: ns1
        matchSetOptions: any
    actions:
      routeDisposition: reject-route
  - name: statement2
    conditions:
      matchPrefixSet:
        prefixSet: ps2
        matchSetOptions: any
      matchNeighborSet:
        neighborSet: ns2
        matchSetOptions: any
    actions:
      routeDisposition: reject-route
```

- example 3
  - If you want to add other policies, just add policy-definitions block
    following the first one like this

```yaml
apiVersion: bgppolicydefinition.loxilb.io/v1
kind: BGPPolicyDefinitionService
metadata:
  name: policy1
spec:
  name: policy1
  statements:
  - name: statement1
    conditions:
      matchPrefixSet:
        prefixSet: ps1
        matchSetOptions: any
      matchNeighborSet:
        neighborSet: ns1
        matchSetOptions: any
    actions:
      routeDisposition: reject-route
---
apiVersion: bgppolicydefinition.loxilb.io/v1
kind: BGPPolicyDefinitionService
metadata:
  name: policy2
spec:
  name: policy2
  - name: statement2
    conditions:
      matchPrefixSet:
        prefixSet: ps2
        matchSetOptions: any
      matchNeighborSet:
        neighborSet: ns2
        matchSetOptions: any
    actions:
      routeDisposition: reject-route
```

- example 4
  - This PolicyDefinition has multiple conditions including BgpConditions as
    follows:
    - prefix-set: *ps1*
    - neighbor-set: *ns1*
    - community-set: *community1*
    - ext-community-set: *ecommunity1*
    - as-path-set: *aspath1*
    - as-path length: *equal 2*
  - If a route matches all these conditions, it will be accepted with community
    "65100:20", next-hop 10.0.0.1, local-pref 110, med subtracted 200, as-path
    prepended 65005 five times.

```yaml
# example 4
apiVersion: bgppolicydefinition.loxilb.io/v1
kind: BGPPolicyDefinitionService
metadata:
  name: example-policy
spec:
  name: example-policy
  statements:
  - name: statement1
    conditions:
      matchPrefixSet:
        prefixSet: ps1
        matchSetOptions: any
      matchNeighborSet:
        neighborSet: ns1
        matchSetOptions: invert
      bgpConditions:
        matchCommunitySet:
          communitySet: community1
          matchSetOptions: any
        matchExtCommunitySet:
          communitySet: ecommunity1
          matchSetOptions: any
        matchAsPathSet:
          asPathSet: aspath1
          matchSetOptions: any
        asPathLength:
          operator: eq
          value: 2
        afiSafiIn:
          - l3vpn-ipv4-unicast
          - ipv4-unicast
    actions:
      routeDisposition: accept-route
      bgpActions:
        setMed: "-200"
        setAsPathPrepend:
          as: "65005"
          repeatN: 5
        setCommunity:
          options: add
          setCommunityMethod:
            communitiesList:
              - 65100:20  
```

### 4. Attaching policy

Here we explain how to attach defined policies to
[neighbor local rib](#42-attach-policy-to-route-server-client).

#### 4.1. Attach policy to route-server-client

You can use policies defined above as Import or Export or In policy by
attaching them to neighbors which is configured to be route-server client.

To attach policies to neighbors, you need to add policy's name to
`neighbors.apply-policy` in the neighbor's setting.
This example attaches *policy1* to Import policy and *policy2* to Export policy
and *policy3* is used as the In policy.

```yaml
apiVersion: bgppolicyapply.loxilb.io/v1
kind: BGPPolicyApplyService
metadata:
  name: policy-apply
spec:
  ipAddress: "10.0.255.2"
  policyType: "import"
  polices:
  - "policy1"
  routeAction: "accept"
```

neighbors has a section to specify policies and the section's name is
apply-policy. The apply-policy has 4 elements.

| Element | Description | Example |
| --- | --- | --- |
| ipAddress | neighbor IP address | "10.0.255.2" |
| policyType | option for the Policy type:<br> "import" or "export" . | "import" |
| polices | The list of the policy |   - "policy1" |
| routeAction | action when the route doesn't match any policy or none of the matched policy specifies route-disposition:<br> "accept" or "reject".  | "accept" |

## Policy and Soft Reset

When you change an import policy and reset the inbound routing table (aka soft reset in), a withdraw for a route rejected by the latest import policies will be sent to peers. However, when you change an export policy and reset the outbound routing table (aka soft reset out), even if a route is rejected by the latest export policies, a withdraw for the route will not be sent.

The outbound routing table doesn't exist for saving memory usage, it's impossible to know whether the route was actually sent to peer or the route also was rejected by the previous export policies and not sent. GoBGP doesn't send such withdraw rather than possible unwilling leaking information.

