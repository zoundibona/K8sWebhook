<div align="center">
  
   ![téléchargement](https://github.com/user-attachments/assets/d7677682-5b84-4c64-88b9-c08cf2268e08)

</div>



# KUBERNETES WEBHOOK 

Admission webhooks are HTTP callbacks that receive admission requests and do something with them. You can define two types of admission webhooks, validating admission webhook and mutating admission webhook. Mutating admission webhooks are invoked first, and can modify objects sent to the API server to enforce custom defaults. After all object modifications are complete, and after the incoming object is validated by the API server, validating admission webhooks are invoked and can reject requests to enforce custom policies.
