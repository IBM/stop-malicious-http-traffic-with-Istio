# Stop malicious http traffic with Istio
It's always been a critical task to secure a your web servcies. In the world of microservices, it's been even more challenging. Many orchastration tools, like Kubernetes, lack the ability to detect and react to malicious attacks, especially at serice/pod level. Forturnetely the emerging of new technologies like Istio makes that task possible. 

In this pattern, the malicous traffic we address is not the tranditional Ddos attacks, which happens usually at layer 3 or 4.  Honestly, it's beyond the capability of either Kubernetes or Istio to deal with millions of requests per second. It is the job of the Infruasture provider to address these problems. The malicious traffic we tackle is normally smaller in scale but more damaging to the services at layer 5-7.

Below are the traffic scenarios this pattern is targeting:
a. The client is using curl to frequently access the service. This is an indiator that the client is a bot than a customer, which normally uses web browser to access the service. Of course, there are other tools like "wget", but here we are just setting an example. Our solution will be if the curl happen too frequently, the service will be denied.    
b. The client is accessing the service too frequently. This scenario is like a simple Ddos attack, where the client will access the services way more than human can process the information.  In this case, we also set a threshold. If the client goes over that limit, the IP address will be banned from accessing the service.   
c. Hacking: The client will try to "guess" the common URI so it can either do sql injection or getting information out of the service. During the prep of this pattern, my cluster was actually attacked. Of course they got nothing but we all know now how real it is. The common symptom is they would try to access url/db or url/sql, and then get a 40x error from the server. The solution is to put a low threshold for 40x errors for each ip and block the ip if it goes over the threshold.

## Istio
Istio project is an joint open source effort by Lyft, Google and IBM. Istio provides better management for service mesh. The benefits of Istio brings in are :
a. Better insight of the cluster
b. Policy enforcement at service/pod level
c. Encrypted traffic between service/pods traffic

Although Istio is not dependent on Kubernetes, its features are best demostrated under Kubernetes. Istio has several components. In this pattern we are focused on Mixer.

Mixer is a customizing tool so that the backend tools, such as Logging and Telemetry display systems can get the tailered information through policy configuration rather than code manipulation.
We choose the Quota adapter in the Mixer because:
a. It supports stateful logic. Quota is counted for a period of time whereis the routing rules are stateless. 
b. It has a feed back system. The exhustion of quota will result in the Deiner adapter to kick in, result in denying the request. Other Mixer adapters, like Prometheus, though provides more visual effects, lack the feed back mechanism.

## Pre-requisits
Install Minikube, follow this link[https://kubernetes.io/docs/getting-started-guides/minikube/].
Install Istio, follow this link[https://istio.io/docs/setup/kubernetes/quick-start.html]
Please choose "Install Istio without enabling mutual TLS authentication" and "Install the Istio-Initializer"
After you are able to point the Kubectl to your cluster, run 
kubectl get nodes
