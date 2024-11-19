# 5G-UPF-Scaling

## Contact

For help and information for this project refer to this email : Sokratis.Christakis@lip6.fr 

## Overview

In this project you are asked to deploy a fully-operational 5G network using OpenAirInterface(OAI) and Kubernetes. Using Kubernetes means that your different network functions will run as independent containers(Docker) within a microservices environment provided by Kubernetes. The goal of this project is to observe the throughput that the UPF is forwarding. Based on that, you are asked to scale the number of UPF instances that deal with these throughput values.

## Getting Started

The first step is to deploy the 5G Core Network.

Clone this repository in your VM to access the files that we have already prepared for you:
```
git clone https://gitlab.com/schristakis1/5g-up-scaling.git
```


Go inside the the folder that you have just cloned and study the files and more specifically the folder oai-5g-core and oai-ueransim.yaml.

- In this project you will have to deploy the 5G Core funtions **in this specific order** : mysql(Database), NRF, UDR, UDM, AUSF, AMF, SMF, UPF.


```
cd 5g-amf-scaling/oai-5g-core
```
In order to deploy each function you have to execute the following command for each core network function:

```
helm install {network_function_name} {path_of_network_function}
```
For example if you want to deploy the sql database  and then the NRF, UDR you execute:

```
helm install mysql mysql/
helm install nrf oai-nrf/
helm install udr oai-udr/
...
```
It is very important that every after helm command you execute: kubectl get pods in order to see that the respective network function is running, before going to the next one.

In order to uninstall something with helm you execute the following command:
```
helm uninstall {function_name}  ## e.g. helm uninstall udr
```

Afer you deployed all the network functions mentioned above you will be able to connect the UE to the 5G network by executing:

```
kubectl apply -f oai-ueransim.yaml
```

In order to uninstall something with kubectl you execute the following command:
```
kubectl delete -f oai-ueransim.yaml 
```

In order to see if the UE has actually subscribed and received an ip you will have to enter the UE container:

```
kubectl get pods # In order to find the name of the ue_container_name (should look something like this: ueransim-746f446df9-t4sgh)
kubectl exec -ti {ue_container_name} -- bash
```

If everything went well you should be inside the UE container and if you execute ifconfig you should see the following interface:
```
uesimtun0: flags=369<UP,POINTOPOINT,NOTRAILERS,RUNNING,PROMISC>  mtu 1400
        inet 12.1.1.2  netmask 255.255.255.255  destination 12.1.1.2
        inet6 fe80::73e2:e6e6:c3a:3d17  prefixlen 64  scopeid 0x20<link>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 500  (UNSPEC)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 12  bytes 688 (688.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

If not, something went wrong. If yes, execute the following command to make sure you have connectivity to the internet through your 5G network:
```
ping -I uesimtun0 8.8.8.8
```


If the ping command works, it means that you have successfully deployed a 5G network and connected a UE to it.

**Extra:** Since you are dealing with throughput, check that iperf works as well:

Enter the UPF container using the following command:
```
kubectl exec -ti {upf_container_name} -- bash       #(get the name again with executing kubectl get nodes)
iperf -s -i 1 -B 12.1.1.*    ## Server (the tun0(12.1.1.*) ip should be visible if you execute ifconfig in the UPF container)
```

Then, enter the UE(client) container and execute:
```
kubectl exec -ti {ue_container_name} -- bash        #(get the name again with executing kubectl get nodes)
iperf -s -i 1 -B 12.1.1.* -c {IP_UPF_FROM_BEFORE} -b 10M        ## Client (the uesimtun0(12.1.1.*) ip should be visible if you execute ifconfig in the UE container)
```
**Caution: Do not test iperf with over 100M because the link will drop and then you will have to recreate the 5G network.**

## Project goal

As previously mentioned you will have to extend this architecture to manually scale the number of UPFs in the network based on throughput values. The throughput values you will have to use in this project are located in the throughput_values.txt file. Originally  your network will have only one UPF function, but if the throughput is getting higher you will have to scale the UPF deployment to deal with evolving throughput demands.

Hint: You will have to generate traffic from the UE to the UPF based on the file values(throughput_values.txt). Then you will have to develop a script that will measure the throughput of the UPF(you should "hear" the tun0 interface in the UPF) and takeaction to manually increase or decrease the UPF instances based on this value that you will retrieve.

The scaling should work as follows:

- if throughput <= 10 Mpbs --> 1 UPF instances
- if throughput 10 < throughput <= 20 Mpbs --> 2 UPF instances
- if throughput 20 < throughput <= 40 Mpbs --> 3 UPF instances
