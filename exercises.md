# Kubernetes: Networking & Security

## Lab access
* SSH: 34.91.200.128
* User: labuser*xx* (i.e. labuser03)
* Pass: P@ssw0rd!123

From this VM you can use ```kubectl``` to access the cluster.

## Networks
| | Subnet | Lab 3 example |
|-|--------|---------------|
| Nodes | 10.x.0.0/24 | 10.3.0.0/24 |
| Pods | 10.1xx.0.0/24 | 10.103.0.0/24 |
| Services | 10.2xx.0/24 | 10.203.0.0/24 |
| FW Internal | 192.168.2xx.2/24 | 192.168.203.2/24 |

*x = lab instance*

## Credentials

### Firewall
* username: labadmin
* password: P@ssw0rd!123

## Excercise 1: Connect and check thee environment
* Connect to the firewall and open the monitor tab.
  * It helps to put a filter in
    ```
    addr.dst in 172.16.0.200 or addr.dst in 1.1.1.1
    ```
  * In the upper-right corner, you can change 'manual' to '10 seconds' (this will automatically refresh the logging)
* Connect to the cluster
* Deploy the 'meetup monitoring/dashboard application'.
  ```
  kubectl create -f https://raw.githubusercontent.com/on2itsecurity/meetup-kubenetsec/master/manifests/tests.yaml
  ```
  *It can take a few seconds before the logging appears*
  Two pods will be deployed, one pod that will run diagnostics and one pod on the receiving end.
  The pod running the diagnostics will connect to;
    * Dashboard VM: 172.16.0.200
    * Internet: 1.1.1.1
    * OtherPod: 10.1xx.x.x
* Did you see all the traffic in the firewall and why ?
* What is the difference between connections to the VM and Internet ?
* What did you expect and did it comply ?

## Exercise 2: Install Cilium

* Install Cilium
  ```bash
  kubectl create -f https://raw.githubusercontent.com/on2itsecurity/meetup-kubenetsec/master/manifests/cilium.yaml
  ```
* Wait till all pods are running
  ```
  kubectl get pods -A
  ```
* Since the CNI has changed we need to restart the pods
  ```
  kubectl delete pod --all-namespaces --all
  ```
* Repeat the same tests as in exercise 1, do you notice any difference ?

## Exercise 3: Disable NAT
* Disable 'masquerade' in the config of Cillium
  ```
  kubectl edit configmap -n cilium cilium-config
  ```
  Change the following line
  ```
  masquerade: "false"
  ```
* Delete the Cillium pods (to load the new config)
  ```
  kubectl delete pod -n cilium -l k8s-app=cilium
  ```
* To overrule the default masquerade set by GKE, we need install the 'no-masq-daemonset'

* Adjust the config file, which tells to NAT / Masquerade nothing.
  ```
  cat <<EOF > config
  nonMasqueradeCIDRs:
  - 0.0.0.0/0
  resyncInterval: 60s
  EOF
  ```
  ```
  kubectl create configmap ip-masq-agent --from-file config -n kube-system
  ```
* Create the deamonset
  ```
  kubectl create -f https://raw.githubusercontent.com/on2itsecurity/meetup-kubenetsec/master/manifests/ip-masq-agent-original-ds.yaml
  ```
* Repeat the same tests as in exercise 1, do you notice any difference ?
* Why is outside traffic not allowed anymore ?
* Why don't we want to allow pods to the Internet ?

## Exercise 4: Install PAN POD Updater

* Since we use a public GKE setup, we need to make an exception by not masquerading the master IP.
* To make sure that the Nodes and Pods can still reach the Master, we need to make an exception for the public IP of the master.
  ```
  kubectl get ep
  ```
  *It is probably in the 34.x.x.x*
* Adjust the config file, which tells to NAT / Masquerade nothing, fill in the obtained public IP.
  ```
  kubectl edit configmap -n kube-system ip-masq-agent
  ```
  ```
  nonMasqueradeCIDRs:
    - 0.0.0.0/0
  masqueradeCIDRs:
    - 34.x.x.x/32
  resyncInterval: 60s
  masqLinkLocal: false
  ```
* Create the deamonset
  ```
  kubectl apply -f https://raw.githubusercontent.com/on2itsecurity/meetup-kubenetsec/master/manifests/ip-masq-agent-ds.yaml
  ```
* Restart the deamonset
  ```
  kubectl delete pod -n kube-system -l k8s-app=ip-masq-agent
  ```
* Generate an API Key to access the firewall, the result is displayed between the ```<result>``` tags.
  ```
  kubectl run getapikey --image=curlimages/curl --restart=Never --rm -ti -- curl -k -X POST 'https://192.168.2xx.2/api/?type=keygen&user=labadmin&password=P@ssw0rd!123'
  ```
* Create the env.ini file and fill in the IP address of the firewall and the API-key.
  ```
  nano env.ini
  ```
  ```
  [PAN-FW]
  Token=LUFRPT1iSEhDekc5VkFqSWxjRmZNS25EMy9EeWNIRm89bE9lMHhKOFVkSmxiWmFhK3E4OFBtcnkzaUhUMm1RcVZabERObmNnazdzQkpVNEV3RGQxWG1DRXR6MzBoSkNMcg==
  URL=https://192.168.203.2
  RegisterExpire=90

  [SYNC]
  Namespace=
  LabelKeys=app
  FullResync=45
  ClearAllRegisteredOnStart=
  ```
* Create a secret from the env.ini file.
  ```
  kubectl create secret generic noc-k8slabels --from-file env.ini
  ```
* Deploy the PAN Pod Updater
  ```
  kubectl create -f https://raw.githubusercontent.com/on2itsecurity/meetup-kubenetsec/master/manifests/noc-k8slabels-v1.yaml
  ```
* The PAN updater will update the labels to firewall in so called 'Dynamic Address Groups'.
* On the firewall go to tab Policies and on the left to security. The first rule has a source-address 'diagnostic-pods', hover over it and click on the arrow on the right. Click on Value, what does Match mean ?
* Click on more, to see the contents of the address-group. (write down or remember the IP address)
* Now we can easily allow this traffic, click on the rule and go to actions, change the action to allow and click Ok.
* On the topright corner click commit.

* To show the effectiveness of working with Dynamic groups we are going to delete the pod now.
  ```
  kubectl delete pod -n diagnostic -l app=runtests
  ```
* If you look within 90 seconds, the group should display two entries now.
* When the TTL expires see env.ini, the 'old' entry dissapears.

## Exercise 5: Generate Intra- pod & cluster traffic

* The runtests pod will connect to the otherpod in an other namespace.
* On the dashboard you should be able to see if this connection is succesful.
* What kind of traffic is going from the testpod to the otherpod ?
  * How would you inspect this traffic ?
  * How would you control this traffic ?

## Exercise 6: Hubble
* Install Hubble
  ```
  kubectl apply -f https://raw.githubusercontent.com/cilium/hubble/master/tutorials/deploy-hubble-servicemap/hubble-all-minikube.yaml
  ```
* Install the Hubble service
  ```
  kubectl create -f https://raw.githubusercontent.com/on2itsecurity/meetup-kubenetsec/master/manifests/hubble-ui-service.yaml
  ```
* Go to the public IP address you received for Hubble.
* Are you able to see the traffic from the runtest pod ?
* What connections are there ?

## Exercise 7: NetworkPolicies

* Since we handle a namespace as a 'Zero Trust' segment, namespaces should not be able to talk to each other, unless we explicitly allow it.
* Create a networkpolicy
  ```
  cat <<EOF | kubectl create -n othernamespace -f -
  apiVersion: "cilium.io/v2"
  kind: CiliumNetworkPolicy
  metadata:
    name: "allow-within-namespace"
  specs:
    - endpointSelector:
        matchLabels: {}
      egress:
      - toEndpoints:
        - matchLabels:
            app: othertestpod
            "k8s:io.kubernetes.pod.namespace": othernamespace
      ingress:
      - fromEndpoints:
        - matchLabels:
            "k8s:io.kubernetes.pod.namespace": othernamespace
  EOF
  ```
  * The above networkpolicy will allow traffic within the namespace and will allow the testpod to go outside (to update the dashboard).

## Exercise 8: Optional Layer 7 policies
We need to figure out together :)

