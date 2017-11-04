# Stop malicious http traffic with Istio
It's always been a critical task to secure a your web services. In the world of microservices, it's been even more challenging. Many orchestration tools, like Kubernetes, lack the ability to detect and react to malicious attacks, especially at service/pod level. Fortunately the emerging of new technologies like Istio makes that task possible. 

In this pattern, the malicious traffic we address is not the traditional Ddos attacks, which happens usually at layer 3 or 4.  Honestly, it's beyond the capability of either Kubernetes or Istio to deal with millions of requests per second. It is the job of the Infrastructure provider to address these problems. The malicious traffic we tackle is normally smaller in scale but more damaging to the services at layer 5-7.

Below are the traffic scenarios this pattern is targeting:
a. The client is using curl to frequently access the service. This is an indicator that the client is a bot than a customer, which normally uses web browser to access the service. Of course, there are other tools like "wget", but here we are just setting an example. Our solution will be if the curl happen too frequently, the service will be denied.    
b. The client is accessing the service too frequently. This scenario is like a simple Ddos attack, where the client will access the services way more than human can process the information.  In this case, we also set a threshold. If the client goes over that limit, the IP address will be banned from accessing the service.   
c. Hacking: The client will try to "guess" the common URI so it can either do sql injection or getting information out of the service. During the prep of this pattern, my cluster was actually attacked. Of course they got nothing but we all know now how real it is. The common symptom is they would try to access url/db or url/sql, and then get a 40x error from the server. The solution is to put a low threshold for 40x errors for each ip and block the ip if it goes over the threshold.

## Istio
Istio project is an joint open source effort by Lyft, Google and IBM. Istio provides better management for service mesh. The benefits of Istio brings in are :
a. Better insight of the cluster
b. Policy enforcement at service/pod level
c. Encrypted traffic between service/pods traffic

Although Istio is not dependent on Kubernetes, its features are best demonstrated under Kubernetes. Istio has several components. In this pattern we are focused on Mixer.

Mixer is a customizing tool so that the backend tools, such as Logging and Telemetry display systems can get the tailored information through policy configuration rather than code manipulation.
We choose the Quota adapter in the Mixer because:
a. It supports statutes logic. Quota is counted for a period of time whereas the routing rules are stateless. 
b. It has a feed back system. The exhaustion of quota will result in the Deiner adapter to kick in, result in denying the request. Other Mixer adapters, like Prometheus, though provides more visual effects, lack the feed back mechanism.

## Pre-requisite
* Install Minikube, follow this [link](https://kubernetes.io/docs/getting-started-guides/minikube/) (Version >1.7.4)           
* Install Istio, follow this [link](https://istio.io/docs/setup/kubernetes/quick-start.html)  (Version Alpha 0.2)      
* Please choose "Install Istio without enabling mutual TLS authentication" and "Install the Istio-Initializer"     
* Make sure your Kubectl points to your cluster
* Run `git clone https://github.com/IBM/stop-malicious-http-traffic-with-Istio.git`  Then `cd stop-malicious-http-traffic-with-Istio` from a terminal and keep that terminal.    

## Install sample app
We use httpbin app, which can print out the basic information of your request. The typical uris are:
http://*url*/ip   and    
http://*url*/headers

From the previous terminal, run `kubectl apply -f sample/httpbin.yaml`
Since we have enabled Istio Initilizer, which will automatically deploy an Envoy sidecar to the pod, no more istioctl injection is needed here.
Also we designated the httpbin service type as LoadBalancer, so we can access the service via its ip address.
To access the service, run `kubectl get svc httpbin -o jsonpath='{.spec.ports[0].nodePort}'` to get the ip of the service. Then from a browser, try `http://*ip*/headers`

## Enable external ip address to be recorded 
From the same terminal, run `kubectl patch svc httpbin -p '{"spec":{"externalTrafficPolicy":"Local"}}' -n istio-system`
By default, the client ip is masqueraded at the node. By applying this patch, the original client ip will be kept in http header.

## Limit access rate per ip
In this config, we limit the access to 3/min per ip. Of course this low rate is for demo purpose. The config has 3 parts: 
* Handler set the concrete quota of 3 access per min.   
* Instance specify the application of the quota. In our case, the original ip address from the client.   
* Rule sets the condition when the handler will be invoked to process the instance. In this case, it is always.
From the same terminal, run `kubeclt apply -f config/ratelimit.yaml`    
Then try to access the service 3 times in a short period of time.
After that, try again. You should see an error message "Quota was exhausted."

## Ban curl command
This config has the same structure as the last one. Of course the rules are different. First we directly call denier adapter and not going through quota since we'll directly deny all incoming requests using curl.   
The second difference is that we do a fuzzy match ` match: match(request.headers["user-agent"], "curl*")` since curl has different versions so any user agent starts with "curl" will be blocked.

Try it out: From the terminal, run `curl http://*ip*/ip`. You should see the client ip address.   
Now run `kubectl apply -f config/curl.yaml`    
Then try again. Your request will be denied.   

## Block phishing uris
This config again uses the quota adapter because we should allow certain amount of URI errors from one IP address. However if is too much, we need to block it.
So the quota handler is applied on each ip address with the same response error. In this example, we choose 404 error. We could also do a fuzzy 40x error. But in reality, the web server will more than likely to respond with 404 error if the resource does not exist.

Run from terminal `kubectl apply -f config/error.yaml`    
Now try in a browser(in case you did not remove the curl ban): `http://*ip*/sql`    
After 3 times, you should get the same denial as the first config.
