# Class Based Forwarding over RSVP LSPs Design Consideration

In my previous write-ups, I covered Class of Service (CoS) design considerations and various other traffic engineering mechanisms, including Faster Auto-Bandwidth Adjustment and Container LSP. While these features address modern traffic engineering challenges, there may still be a business need to classify traffic such that high-priority or mission-critical traffic travels via the shortest or best path, medium-priority traffic follows the second-best path, and low-priority traffic is routed via the longest link.

Class-Based Forwarding (CBF) addresses this challenge by directing traffic of each specific forwarding class to label-switched paths that align with business requirements. This solution assumes traffic classification occurs either through behavior aggregate or multi-field classifierer and traffic directed to respective forwarding queue on egress interfaces. Label-switched paths are mapped to these forwarding classes, ensuring that certain types of traffic are transmitted through designated egress interface queues and specific label-switched paths.

## Solution Brief 

![topology](./images/topology.png)

Let’s suppose Site-A needs to communicate with Site-B. As shown in the topology diagram above, there are three distinct paths available between these sites. The direct path between Site-A and Site-B is the shortest and optimal path in terms of latency, making it the ideal candidate for handling mission-critical traffic within high-priority Label-Switched Paths (LSPs). The path passing through Site-C is the second-best option, making it suitable for handling medium-priority traffic within "Medium" forwarding class LSPs. Finally, the path passing through Site-D has the highest latency and is best suited for carrying lower-priority traffic within the Best-Effort forwarding class.

This approach ensures each traffic class is efficiently directed according to its priority and latency requirements, aligning with Class-Based Forwarding principles to optimize path selection based on business-driven performance needs.

## Solution Components  
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
In Junos, only the next-hops associated with the "active-path" are considered for the next-hop-map. Therefore, it is crucial that each forwarding class referenced in the next-hop-map has a corresponding LSP under the 'active-path' output. However, if multipath is enabled and the same subnet is learned from multiple remote PEs, how will the forwarding classes be mapped to the LSPs from all remote PEs installed in both the control and forwarding planes as next-hops for the subnet in question?

In this scenario, each forwarding class must be aligned with the appropriate LSP based on routing policies, metrics, and other attributes. This can be achieved by leveraging regular expressions in the next-hop-map configuration, allowing the router to dynamically match and select the correct LSP for each forwarding class, even when multiple remote PEs advertise the same destination. Thus, effective route selection and traffic classification can be maintained across different LSPs from various remote PEs.

### Indexd Next Hop

For each forwarding class referenced in the next-hop-map,  corresponding LSP/ LSPs are selected either through regular expressions or by matching specific names. An indexed next-hop is then created in the Forwarding Information Base (FIB), facilitating efficient packet forwarding based on the defined forwarding classes and their associated LSPs.

To view the forwarding-class mapping with corresponding LSPs, we first need to obtain the Queue ID for the respective forwarding class.
```
show class-of-service forwarding-class 
Forwarding class                       ID      Queue     No-Loss
  BEST-EFFORT                          0         0       disabled
  MISSION-CRITICAL                     2         2       disabled
  NETWORK-CONTROL                      3         3       disabled
  SCAVENGER                            4         5       disabled
  VIDEO                                5         4       disabled
  VOICE                                1         1       disabled
```
Once the Queue IDs are known, we can identify the indexed next-hop for each forwarding class. For example, index:1 indicates that the VOICE forwarding class is mapped to four unicast next-hops (NHS). Recall from our next-hop-map configuration that we mapped the VOICE queue to LSPs matching the regex .*-HPRI-.*. There are two LSPs that match this regex: right-side-PE1-to-left-side-PE1-HPRI-LSP and right-side-PE1-to-left-side-PE2-HPRI-LSP, with one pointing to left-side-PE1 and the other pointing to left-side-PE2. The remaining two NHS for index:1 are associated with backup LSPs. For the Best-Effort or default forwarding class, or any forwarding class not explicitly referenced in the next-hop-map, an idx:xx entry will be created. If only the Best-Effort or lowest-priority forwarding class is referenced in the next-hop-map, an indexed next-hop will not be created.

A detailed explanation of the Junos FIB and next-hop types is beyond the scope of this document.
```
show route forwarding-table destination 172.172.21.0/24 
Routing table: trg.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
172.172.21.0/24    user     0                    ulst     8858     1
                                                 comp     8421     1
                                                 indr     8420     1
                                                 idxd     8876     1
                   idx:1                         ulst     8873     1
                                                sftw Push 4660     8736     1 et-0/2/6.0
                              10.10.10.116       ucst     3004     1 et-0/2/6.0
                                                sftw Push 349     8806     1 ae0.0
                              10.10.10.114       ucst     3001     1 ae0.0
                   idx:2                         ulst     8873     1
                                                sftw Push 4660     8736     1 et-0/2/6.0
                              10.10.10.116       ucst     3004     1 et-0/2/6.0
                                                sftw Push 349     8806     1 ae0.0
                              10.10.10.114       ucst     3001     1 ae0.0
                   idx:5                         ulst     8874     1
                                                sftw Push 4659     8737     1 et-0/2/6.0
                              10.10.10.116       ucst     3004     1 et-0/2/6.0
                                                sftw Push 349     8807     1 ae0.0
                              10.10.10.114       ucst     3001     1 ae0.0
                   idx:xx                        ulst     8875     1
                                                sftw Push 4656     8681     1 et-0/2/6.0
                              10.10.10.116       ucst     3004     1 et-0/2/6.0
                                                sftw Push 288     8694     1 ae0.0
                              10.10.10.114       ucst     3001     1 ae0.0
                                                 comp     8817     1
                                                 indr     8816     1
                                                 idxd     8866     1
                   idx:1                         ulst     8861     1
                                                sftw Push 4663     8812     1 et-0/2/6.0
                              10.10.10.116       ucst     3004     1 et-0/2/6.0
                                                sftw Push 286     8859     1 ae0.0
                              10.10.10.114       ucst     3001     1 ae0.0
                   idx:2                         ulst     8861     1
                                                sftw Push 4663     8812     1 et-0/2/6.0
                              10.10.10.116       ucst     3004     1 et-0/2/6.0
                                                sftw Push 286     8859     1 ae0.0
                              10.10.10.114       ucst     3001     1 ae0.0
                   idx:5                         ulst     8862     1
                                                sftw Push 4662     8813     1 et-0/2/6.0
                              10.10.10.116       ucst     3004     1 et-0/2/6.0
                                                sftw Push 4662, Push 346(top)     8860     1 et-0/2/7.0
                              10.10.10.92        ucst     3007     1 et-0/2/7.0
                   idx:xx                        ulst     8865     1
                                                sftw Push 4661     8814     1 et-0/2/6.0
                              10.10.10.116       ucst     3004     1 et-0/2/6.0
                                                sftw Push 4661, Push 346(top)     8864     1 et-0/2/7.0
                              10.10.10.92        ucst     3007     1 et-0/2/7.0
```
## Conclusion 
Class-Based Forwarding (CBF) is an effective component that introduces an additional layer of traffic engineering, enabling the differentiation of traffic based on business needs. It allows low-priority traffic to traverse slower paths while ensuring that business-critical traffic utilizes the fastest or best available paths. However, there are some downsides to consider. In a brownfield environment, it is essential to clearly define LSP names prior to implementing CBF to ensure that the mapping between forwarding classes and LSPs is achievable.

