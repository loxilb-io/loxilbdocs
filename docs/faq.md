# loxilb FAQs

* <b>Does loxilb depend on what kind of CNI is deployed in the cluster ?</b>

Yes, loxilb configuration and operation might be related to which CNI (Calico, Cilium etc) is in use. loxilb just needs a way to find a route to its end-points. This also  depends on how the network topology is laid out. For example, if a separated network for nodePort and external LB services is in effect or not. We will have a detailed guide on best practices for loxilb deployment soon. In the meantime, kindly reach out to us via github or [loxilb forum](www.loxilb.io)

* <b>Can loxilb be possibly run outside the released docker image ?</b>

Yes, loxilb  can be run outside the provided docker image. Docker image gives it good portability across various linux like OS's without any performance impact. However, if need is to run outside its own docker, kindly follow README of various loxilb-io repositories.

* <b>Can loxilb also act as a CNI ?</b>

loxilb supports all functionalities of a CNI but loxilb dev team is happy solving external LB and connectivity problems for the time being. If there is a future requirement, we might work on this as well

* <b>Is there a commercially supported version of loxilb ?</b>

At this point of time, loxilb-team is working hard to provide a high-quality open-source product. If users need commercial support, kindly get in touch with us

* <b>Can loxilb run in a standalone mode (without Kubernetes) ?</b>

Very much so. loxilb can run in a standalone mode. Please follow various guides available in loxilb repo to run loxilb in a standalone mode.






