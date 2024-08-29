<div align="center">
  
   ![téléchargement](https://github.com/user-attachments/assets/d7677682-5b84-4c64-88b9-c08cf2268e08)

</div>





# KUBERNETES WEBHOOK 

Admission webhooks are HTTP callbacks that receive admission requests and do something with them. You can define two types of admission webhooks, validating admission webhook and mutating admission webhook. Mutating admission webhooks are invoked first, and can modify objects sent to the API server to enforce custom defaults. After all object modifications are complete, and after the incoming object is validated by the API server, validating admission webhooks are invoked and can reject requests to enforce custom policies. <br>
Webook provide a security layer as it can deny some kubernetes objects to be created if they do not some criterias like the image and labels, ...
An example could be a situation where you can predefined a list of allowed images, if the pod image to be created is not part of the list then pod creation will be rejected.
Here in this case I will be using a custom script developed in Python Flash to mutate and validate kubernetes pod objects creation

# REQUIREMENTS

There two ways to run the webhook server, you can run as external server like I did or run it as POD running within the same cluster
For this setup the webhook is running as external server.
When the API server receives a request the API server will trigger the webhook running the flask script, the API server as of 1.30 version requires https to reach the webhook server.

It means that the server must have private key and SSL certificate.
The API server will also need the CA certificate to verify the server certificate.
So here a CA certificate and key have been created and used to sign server certificate

* Python Flash Script (Version 3.0.3)
* Server certificate and key
* CA certificate and Key to sign server certificate
* Kubernetes cluster (I used 1.30)
  


Kindly note that kubernetes does not accept CN (Common Name) certificate but rather SAN (Subject Alternative Name)
In case you can look over the internet to know the differences between the two certificates you can search over Internet



# SETUP

Kindly check into the github repository for the flask script called (webhookserver.py), the same script does both mutation and validation, for mutation the path is server.webhook.com:5000/mutate
for the validation the path is server.webhook.com:5000/validate. <br>
The server.webhook.com is my webhook server name, it is not public, a static DNS entry has been added in Kubernetes core dns to resolve the dns name to static IP of the device hosting the server.
In case you want to host over Internet which I do not advise make sure you can a public domain.

You can check the file mutate-webhook.yaml to see the configuration of the Mutating Webhook Configuration and validate-webhook.yaml for the Validating Webhook Configuration.
Below is sample of the webhook configuration file. 
The validating and mutating configurations files are similars, be careful of the kind in the manifest file as they are not the same


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
    &nbsp;  &nbsp; &nbsp;  &nbsp; **caBundle:**  LS0tLS.......  &nbsp;  &nbsp; &nbsp;  &nbsp; **This is the CA bundle certificate used to sign the webhook server url, this has to be base64 encoded** <br>   
  &nbsp;&nbsp; &nbsp; **admissionReviewVersions:** ["v1"] <br>
  &nbsp;&nbsp; &nbsp; **sideEffects:** None <br>
  &nbsp;&nbsp; &nbsp; **timeoutSeconds:** 5 <br>

  ---
When this configuration is applied whenever a pod is created the API will reach the webhook server


## CREATION OF CA CERTIFICATE AND SERVER CERTIFICATE


### GENERATE CA KEY AND CERTIFICATE
   openssl genrsa 2048 | tee ca-key.pem     **This will generate private key for the CA**
   
   openssl req -new -x509 -nodes -days 365000 -key caKey.pem   -out caCert.pem  **This will generate a CA certificate signed with private key created in the previous step**


### GENERATE SERVER KEY AND CERTIFICATE
Kubernetes 1.30 requires SAN certificate, the server.conf file used to create SAN certificate is available in the github repository 

$ openssl genrsa -out server.key 2048     &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp;    **This will generate server private key** <br>
$ openssl req -new -key server.key -out server.csr -config server.conf   &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp;  **This will generate a CSR (Certificate Signing Request)    needed to get the server certificate**  <br>
$ openssl x509 -req -in server.csr -CA caCert.pem -CAkey caKey.pem -CAcreateserial -out server.crt -days 100000 -extensions v3_req -extfile server.conf    &nbsp;&nbsp; &nbsp; &nbsp;&nbsp; &nbsp; **This will sign the certificate with CA certificate** <br>

At the end of this step, you will get CA cert and private key, server cert and private.
The CA private key is not needed in kubernetes manifest file, only the CA cert is used to verify the server certificate

## CHANGES IN DNS

I am using server.webhook.com as the server dns name, since the API server must be able to resolve the dns name into an IP, I have added an entry in kubernetes core dns
config map

run kubectl edit -n kube-system cm coredns to edit add a static entry. 
See below file you will find a host entry that resolve the dns name to IP

apiVersion: v1
data:
  Corefile: |2

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


## APPLY THE WEBHOOK CONFIG FILES

**$ kubectl apply -f mutate-webhook.yaml**    <br>
mutatingwebhookconfiguration.admissionregistration.k8s.io/mutate-webhook created <br>
**$ kubectl apply -f validate-webhook.yaml** <br>
validatingwebhookconfiguration.admissionregistration.k8s.io/validate-webhook created <br>

## TEST OF WEBHOOK SERVER

After applying the mutating and validating file, let us test the webhook server <br>
The validating requires the image of the POD to be part of list("redis", "nginx, "httpd")  <br>


**$ kubectl run testpod --image=nginx**   <br>
   pod/testpod created <br>

Let us see if resources requests, limits and serviceaccount fields have changed as expected <br>

**$ kubectl get pods -o yaml**  <br>

I have decided to only show the spec section as this where the changes have been made


  spec:      
   
    containers:   
    - image: nginx   
      imagePullPolicy: Always   
      name: testpod   
      resources:    
        limits:      **resources limits has been added**
          cpu: "2"        
          memory: 256Mi   
        requests:      **resources requests has been added**
          cpu: 500m   
          memory: 128Mi   
  
   
    serviceAccount: sa         default serviceaccount has been chnaged to sa
    serviceAccountName: sa      default serviceaccount has been chnaged to sa

As you can see the mutating webhook has performed some changes and validating has validated the pod creation  <br>
Let us test a use case where the image is not part of the list, let us a ubuntu image which is not part of the allowed list in the Flash script  <br>

  
**$ kubectl run ubuntu --image=ubuntu**  <br>
 Error from server: admission webhook "validate-webhook.test.com" denied the request: **IMAGE(S) NOT IN ALLOWED IMAGES LIST**  <br>

 As you can the validating has denied the creation of the pod as ubuntu has not been added into the list in the Flask script  <br>

  
 
    














