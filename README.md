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

You can check the file mutate-webhook.yaml to see the configuration of the Mutating Webhook Configuration and validate-webhook.yaml for the Validating Webhook Configuration 


**apiVersion**: admissionregistration.k8s.io/v1 <br>
**kind**: ValidatingWebhookConfiguration <br>
metadata: <br>
  name: validate-webhook <br>
webhooks: <br>
- name: validate-webhook.test.com <br>
  rules: <br>
  - apiGroups:   [""] <br>
    apiVersions: ["v1"] <br>
    operations:  ["CREATE"] <br>
    resources:   ["pods"] <br>
    scope:       "Namespaced" <br>
  clientConfig:
    url: https://server.webhook.com:5000/validate <br>
    caBundle: <br>
  admissionReviewVersions: ["v1"] <br>
  sideEffects: None <br>
  timeoutSeconds: 5 <br>








