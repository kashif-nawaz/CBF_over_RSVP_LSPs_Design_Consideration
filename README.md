# Class Based Forwarding over RSVP LSPs Design Consideration

In my previous write-ups, I covered Class of Service (CoS) design considerations and various other traffic engineering mechanisms, including Faster Auto-Bandwidth Adjustment and Container LSP. While these features address modern traffic engineering challenges, there may still be a business need to classify traffic such that high-priority or mission-critical traffic travels via the shortest or best path, medium-priority traffic follows the second-best path, and low-priority traffic is routed via the longest link.

Class-Based Forwarding (CBF) addresses this challenge by directing traffic of each specific forwarding class to label-switched paths that align with business requirements. This solution assumes traffic classification occurs either through behavior aggregate or multi-field classifierer and traffic directed to respective forwarding queue on egress interfaces. Label-switched paths are mapped to these forwarding classes, ensuring that certain types of traffic are transmitted through designated egress interface queues and specific label-switched paths.

## Solution Brief 

![topology](./images/topology.png)

Let’s suppose Site-A needs to communicate with Site-B. As shown in the topology diagram above, there are three distinct paths available between these sites. The direct path between Site-A and Site-B is the shortest and optimal path in terms of latency, making it the ideal candidate for handling mission-critical traffic within high-priority Label-Switched Paths (LSPs). The path passing through Site-C is the second-best option, making it suitable for handling medium-priority traffic within "Medium" forwarding class LSPs. Finally, the path passing through Site-D has the highest latency and is best suited for carrying lower-priority traffic within the Best-Effort forwarding class.

This approach ensures each traffic class is efficiently directed according to its priority and latency requirements, aligning with Class-Based Forwarding principles to optimize path selection based on business-driven performance needs.

## Solution Componentt 
It is assumed that either Behavior Aggregate or Multifield Classification is applied to incoming traffic at the Site-A Provider Edge (PE) router, assigning traffic to the appropriate forwarding queue on the egress interface. Label-Switched Paths (LSPs) are configured between Site-A and Site-B, using link-coloring or administrative groups to enforce specific LSPs to traverse designated links or paths.

### CoS Forwarding Policy
We need to configure a next-hop-map that maps forwarding classes to LSPs either by specifying exact LSP names or by using regular expressions. Using specific LSP names in the next-hop-map is practical when there is only a single remote PE advertising destination subnets. However, if multiple remote PEs are advertising destination subnets and distinct LSP names exist for each remote PE, regex is required. The regex should be configured to match each LSP dynamically for each remote PE, ensuring proper forwarding class alignment with the respective LSPs towards each remote PE.  

```
class-of-service {
    forwarding-policy {
        next-hop-map CBF {
            forwarding-class BEST-EFFORT {
                lsp-next-hop .*-LPRI-.*;
            }
            forwarding-class MISSION-CRITICAL {
                lsp-next-hop .*-HPRI-.*;
            }
            forwarding-class VIDEO {    
                lsp-next-hop .*-MPRI-.*;
            }
            forwarding-class VOICE {
                lsp-next-hop .*-HPRI-.*;
            }
        }
    }
}
```

We need to ensure that the forwarding policy configured above is applied under routing-options.
``` 
routing-options {
    autonomous-system 64980;
    forwarding-table {
        export CBF ;
    }
}
```
### Consideration for LSP Regex 

In Junos, a single best path is selected and installed as the "active-path" towards remote destinations, even when multipath is enabled and multiple next-hops are available in both the control and forwarding planes towards remote PEs. Let’s explore this behavior with example output from the Right-side PE, referring to the topology depicted above.

Below appended output shows that subnet 172.172.21.0/24 is poiting towards 3 LSPs towards "left-side-PE1" and 3 LSPs towards "left-side-PE2". 
```
show route 172.172.21.0/24 table trg.inet.0 protocol multipath                            

prod.inet.0: 10 destinations, 96 routes (9 active, 0 holddown, 36 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

172.172.21.0/24    #[Multipath/255] 00:03:30, metric2 2100
                    >  to 10.10.10.116 via et-0/2/6.0, label-switched-path right-side-PE1-to-left-side-PE1-LPRI-LSP
                       to 10.10.10.116 via et-0/2/6.0, label-switched-path right-side-PE1-to-left-side-PE1-MPRI-LSP
                       to 10.10.10.116 via et-0/2/6.0, label-switched-path right-side-PE1-to-left-side-PE1-HPRI-LSP
                       to 10.10.10.114 via ae0.0, label-switched-path Bypass->10.10.10.116->10.10.10.109
                       to 10.10.10.114 via ae0.0, label-switched-path Bypass->10.10.10.116->10.10.10.1
                       to 10.10.10.114 via ae0.0, label-switched-path Bypass->10.10.10.116->10.10.10.1
                       to 10.10.10.116 via et-0/2/6.0, label-switched-path right-side-PE1-to-left-side-PE2-LPRI-LSP
                       to 10.10.10.116 via et-0/2/6.0, label-switched-path right-side-PE1-to-left-side-PE2-MPRI-LSP
                       to 10.10.10.116 via et-0/2/6.0, label-switched-path right-side-PE1-to-left-side-PE2-HPRI-LSP
                       to 10.10.10.92 via et-0/2/7.0, label-switched-path Bypass->10.10.10.116
                       to 10.10.10.92 via et-0/2/7.0, label-switched-path Bypass->10.10.10.116
                       to 10.10.10.114 via ae0.0, label-switched-path Bypass->10.10.10.116->10.10.10.5
```

In Junos, the active route (or best path) is  selected by router to reach a specific destination. In Juniper devices, even if multiple paths to a destination are available and multipath routing is enabled, only the best path, marked as the "active-path," is selected for routing and forwarding.
```
show route 172.172.21.0/24 active-path                             

trg.inet.0: 10 destinations, 96 routes (9 active, 0 holddown, 36 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

172.172.21.0/24    @[BGP/170] 00:20:38, localpref 100, from 10.20.48.5
                      AS path: 65002 I, validation-state: unverified
                    >  to 10.10.10.116 via et-0/2/6.0, label-switched-path right-side-PE1-to-left-side-PE1-LPRI-LSP
                       to 10.10.10.116 via et-0/2/6.0, label-switched-path right-side-PE1-to-left-side-PE1-MPRI-LSP
                       to 10.10.10.116 via et-0/2/6.0, label-switched-path right-side-PE1-to-left-side-PE1-HPRI-LSP
                       to 10.10.10.114 via ae0.0, label-switched-path Bypass->10.10.10.116->10.10.10.109
                       to 10.10.10.114 via ae0.0, label-switched-path Bypass->10.10.10.116->10.10.10.1
                       to 10.10.10.114 via ae0.0, label-switched-path Bypass->10.10.10.116->10.10.10.1
```
In Junos, only the next-hop associated with the "active path" is considered for the next-hop-map. Therefore, it is crucial that each forwarding class referenced in the next-hop-map has a corresponding LSP under the 'active path' output. However, if multipath is enabled and the same subnet is learned from multiple remote PEs, how will the forwarding classes be mapped to the LSPs from all remote PEs installed in both the control and forwarding planes as next-hops for the subnet in question?

In this scenario, each forwarding class must be aligned with the appropriate LSP based on routing policies, metrics, and other attributes. This can be achieved by leveraging regular expressions in the next-hop-map configuration, allowing the router to dynamically match and select the correct LSP for each forwarding class, even when multiple remote PEs advertise the same destination. Thus, effective route selection and traffic classification can be maintained across different LSPs from various remote PEs.
