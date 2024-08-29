<div align="center">
  
   ![téléchargement](https://github.com/user-attachments/assets/d7677682-5b84-4c64-88b9-c08cf2268e08)

</div>





# KUBERNETES WEBHOOK 

Admission webhooks are HTTP callbacks that receive admission requests and do something with them. You can define two types of admission webhooks, validating admission webhook and mutating admission webhook. Mutating admission webhooks are invoked first, and can modify objects sent to the API server to enforce custom defaults. After all object modifications are complete, and after the incoming object is validated by the API server, validating admission webhooks are invoked and can reject requests to enforce custom policies. <br>

Webook provide a security layer as it can deny some kubernetes objects to be created if they do not some criterias like the image and labels, ...
An example could be a situation where you can predefined a list of allowed images, if the pod image to be created is not part of the list then pod creation will be rejected.
Here in this case I will be using a custom script developed in Python Flash to mutate the pod details and validate pod creation.

When the API server reaches the webhook it expects a reponse which contains a field **Allow** .
If this field is set to **Yes** then the request is allowed and Kubernetes can mutate/validate the request, otherwise the request is rejected by the API server

# REQUIREMENTS

There are two ways to run the webhook server, you can run it as external server like I did or run it as POD running within the same cluster.
For this setup the webhook is running as external server. 
As of Kubernetes 1.30 the API server requires https to reach the server.
It means that the server must have SSL certificate.
The API server will also need the CA certificate to verify the server certificate.  <br>


below a summary of the requirements : <br>

* Python Flash Script (Version 3.0.3)
* Server certificate and key
* CA certificate and Key to sign server certificate
* Kubernetes cluster (Kubernetes version used : 1.30)
  

Kindly note that kubernetes does not accept CN (Common Name) certificate but rather SAN (Subject Alternative Name)
In case you want to know the differences between CN and SAN you can search over the internet.



# SETUP

Kindly check into the github repository for the flask script called **(webhookserver.py)**, the same script does both mutation and validation, for mutation the path is **server.webhook.com:5000/mutate** while for the validation the path is **server.webhook.com:5000/validate**. <br>

When there is an attempt to create a POD, the request is sent to the API server because both mutating and validating are applied, the request goes first through the mutating step. <br>
At this stage the Flask script will change some fields like serviceAccountName, and also add resources requests and limits to the containers.
After completing the mutating stage, it goes to now to validating stage, if the containers in the POD have images in the list defined in the script then it will be created otherwise it will be rejected

The server.webhook.com is the webhook server name, it is not public, a static DNS entry has been added in Kubernetes Core DNS to resolve the DNS name to static IP of the device hosting the server.
In case you want to host over Internet which I do not advise make sure you have a public domain.

You can check the file mutate-webhook.yaml to see the configuration of the Mutating Webhook Configuration and validate-webhook.yaml for the Validating Webhook Configuration, they are available in this github repository  <br>

Below is sample of a webhook configuration file.   <br>

---

**apiVersion:** admissionregistration.k8s.io/v1 <br>
**kind:** ValidatingWebhookConfiguration  &nbsp;  &nbsp; &nbsp; &nbsp; **The kind will change to MutatingWebhookConfiguration for mutating manifest file**  <br>
**metadata:** <br>
 &nbsp;&nbsp; name: validate-webhook <br>
**webhooks:** <br>
 &nbsp;&nbsp; - name: validate-webhook.test.com <br>
 &nbsp;&nbsp;&nbsp;  rules: <br>
&nbsp;&nbsp;&nbsp; - apiGroups:   [""] <br>
 &nbsp; &nbsp;&nbsp;&nbsp; apiVersions: ["v1"] <br>
 &nbsp; &nbsp;&nbsp;&nbsp; operations:  ["CREATE"] <br>
   &nbsp;&nbsp;&nbsp; resources:   ["pods"] <br>
   &nbsp;&nbsp;&nbsp; scope:       "Namespaced" <br>
&nbsp;&nbsp;&nbsp;   **clientConfig:**  <br>
   &nbsp;  &nbsp; &nbsp;  &nbsp; **url:** https://server.webhook.com:5000/validate  &nbsp;  &nbsp; &nbsp;  &nbsp;  **This is the webhook server url to be reached**  <br>
    &nbsp;  &nbsp; &nbsp;  &nbsp; **caBundle:**  LS0tLS.......  &nbsp;  &nbsp; &nbsp;  &nbsp;&nbsp;  &nbsp; &nbsp;  &nbsp;  **This is the CA bundle certificate used to sign the webhook server url, this has to be base64 encoded** <br>   
  &nbsp;&nbsp; &nbsp; **admissionReviewVersions:** ["v1"] <br>
  &nbsp;&nbsp; &nbsp; **sideEffects:** None <br>
  &nbsp;&nbsp; &nbsp; **timeoutSeconds:** 5 <br>

  ---
When this configuration is applied whenever a pod is created the API will reach the webhook server


## &nbsp; STEP1 : CREATION OF CA CERTIFICATE AND SERVER CERTIFICATE


* ### &nbsp;&nbsp;&nbsp;&nbsp;&nbsp; GENERATE CA KEY AND CERTIFICATE


openssl genrsa 2048 | tee caKey.pem    &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp;  **This will generate private key for the CA**  <br>
openssl req -new -x509 -nodes -days 365000 -key caKey.pem   -out caCert.pem   &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp;  **This will generate a self signed certificate** <br>


* ### &nbsp;&nbsp;&nbsp;&nbsp;&nbsp; GENERATE SERVER KEY AND CERTIFICATE
Kubernetes 1.30 requires SAN certificate, the server.conf file used to create SAN certificate is available in the github repository 

$ openssl genrsa -out server.key 2048     &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp;    **This will generate server private key** <br>

$ openssl req -new -key server.key -out server.csr -config server.conf   &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp;  **This will generate a CSR (Certificate Signing Request) <br> &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;needed to get the server certificate**  <br>
<br>
$ openssl x509 -req -in server.csr -CA caCert.pem -CAkey caKey.pem \\    &nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp; **This will sign the certificate with CA certificate**<br>
-CAcreateserial \\ <br>
-out server.crt -days 100000 -extensions v3_req -extfile server.conf       <br>

At the end of this step, you will get CA certificate and private key, server certificate and private key.
The server certificate and private key are required in the Flask script to setup  https server.
The CA private key is not needed in kubernetes manifest file, only the CA certificate is used to verify the server certificate

## &nbsp; STEP 2:  CHANGES IN KUBERNETES CORE DNS

I am using server.webhook.com as the server DNS name, since the API server must be able to resolve the DNS name into an IP, I have added an entry in kubernetes core DNS
config map

run the below command:  <br>
kubectl edit -n kube-system cm coredns <br>

This will edit the configmap used for Kubernetes core DNS. 
You will find a host entry in below output that resolves the dns name(server.webhook.com) to IP (192.168.1.4 which is IP of webhook server)




    .:53 {

       log
       errors
       health {
          lameduck 5s
       }
       ready
       kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
          ttl 30
       }
       prometheus :9153

       hosts {
           192.168.1.4 server.webhook.com
           fallthrough
        }


To test this entry, you use a POD that supports shell, like nginx. Install dns-utils and try nslookup "dns.name". 
If the DNS is able to resolve then it means it works. It is also possible in Webhook manifest file to use the static IP instead of the DNS name
##  &nbsp; STEP 3 :APPLY THE WEBHOOK CONFIG FILES

**$ kubectl apply -f mutate-webhook.yaml**    <br>
mutatingwebhookconfiguration.admissionregistration.k8s.io/mutate-webhook created <br>
**$ kubectl apply -f validate-webhook.yaml** <br>
validatingwebhookconfiguration.admissionregistration.k8s.io/validate-webhook created <br>

##  &nbsp; STEP 4: TEST OF WEBHOOK SERVER

After applying the mutating and validating file, let us test the webhook server <br>
The validating requires the image of the POD to be part of list("redis", "nginx, "httpd")  <br>

* ### SCENARIO 1

In this scenario we will use a case where the POD image is part of the list ("redis", "nginx, "httpd") as defined in the Flask script

**$ kubectl run testpod --image=nginx**   <br>
   pod/testpod created 

Now that the POD has been created, let us see if resources requests, limits and serviceaccount fields have been modified as expected, let us run the below command

**$ kubectl get pods -o yaml**  <br>

I have decided to only show the spec section as this is where the changes have been made by the script


  spec:      
   
    containers:   
    - image: nginx   
      imagePullPolicy: Always   
      name: testpod   
      resources:    
        limits:      **resources limits have been added**
          cpu: "2"        
          memory: 256Mi   
        requests:      **resources requests have been added**
          cpu: 500m   
          memory: 128Mi   
  
   
    serviceAccount: sa         **default serviceaccount has been chnaged to sa**
    serviceAccountName: sa      **default serviceaccountName has been chnaged to sa**

As you can see the mutating webhook has performed some changes, the serviceAccountName and resources were not specified when the POD was created but at the end they have been added. After the mutating stage the validating took place to validate the pod creation.  <br>
It is also possible to use yaml definition file (declarative mode) to create the POD instead imperative mode used in this case




* ### SCENARIO 2
Let us test a use case where the image is not part of the list, let us a ubuntu image which is not part of the allowed list in the Flash script  <br>  

**$ kubectl run ubuntu --image=ubuntu**  <br>
 Error from server: admission webhook "validate-webhook.test.com" denied the request: **IMAGE(S) NOT IN ALLOWED IMAGES LIST**  <br>
 <br> 
 As you can see the validating has denied the creation of the pod as ubuntu has not been added into the list in the Flask script  <br>
 The message returned by webhook is **IMAGE(S) NOT IN ALLOWED IMAGES LIST**, this message can be customized

  
 
    














