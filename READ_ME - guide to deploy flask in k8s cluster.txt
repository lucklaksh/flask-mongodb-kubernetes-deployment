Prerequisites
-------------
	Kubernetes Cluster: Ensure you have a working Kubernetes cluster.
			I am using self managed kubernetes cluster which was configured using kubeadm, as it production grade setting.
	Docker: Required to build and push container images.

Container Images:
-----------------

Webapp Image: lakshmanlucky821/flask-mongodb-app:latest
MongoDB Image: mongo:latest

Inside Kubernetes cluster
------------------------

kubectl get nodes
ubuntu@ip-172-31-2-172:~$ kubectl get nodes
NAME              STATUS   ROLES           AGE     VERSION
ip-172-31-0-192   Ready    <none>          3d21h   v1.30.3
ip-172-31-2-172   Ready    control-plane   3d21h   v1.30.3
ip-172-31-5-192   Ready    <none>          3d21h   v1.30.3

1 Master node
2 Worker nodes

Deployment Architecture
-----------------------
Web Application:

	Deployment: webapp-deployment.yml - Deploys the Flask application.
	Service: webapp-service.yml - Exposes the Flask application via a ClusterIP service.

MongoDB:

	StatefulSet: db-statefulset.yml - Manages MongoDB pods with persistent storage.
	Headless Service: db-headless.yml - Provides direct access to MongoDB pods.
	Persistent Volume: pv.yml - Configures persistent storage for MongoDB.

Ingress:

	Configuration: ingress.yml - Routes external traffic to the Flask application service. Uses app.example.com for routing.


Resource Creation:
-----------------
create all these resources using "kubectl apply -f resource_file.yml"
kubectl apply -f webapp-deployment.yml
kubectl apply -f webapp-service.yml
kubectl apply -f db-statefulset.yml
kubectl apply -f db-headless.yml
kubectl apply -f pv.yml
kubectl apply -f ingress.yml


Nginx-Ingress-controller
-----------------------
I have confiured my own Nginx-Ingress-controller to respond to only ingress resource with Ingress-class as "nginx-trial"

Since I don't have domain name, I have configured the ingress resource as "app.example.com"
so In our local DNS file, "/etc/hosts"  add 
	<loadbalancer-IP>   app.example.com


since its internal dns routing, 

Autoscaling
-----------

kubectl autoscale deployment flask-mongodb-deployment --cpu-percent=70 --min=2 --max=5


GET Method
----------
1. curl http://app.example.com/
   routes to webapp-service, which loadbalances to the deployment application, i.e, webApplication to the path: "/"

ubuntu@ip-172-31-2-172:~/assessment2$ curl http://app.example.com
Welcome to the Flask app! The current time is: 2024-08-08 12:01:32.035376

2. curl http://app.example.com/data 
   routes to webapp-service, which loadbalances to the deployment application, i.e, webApplication to the path: "/data" , which points to headless db service
      dbservice retreives the data from the database inside the mongodb Pod.

ubuntu@ip-172-31-2-172:~/assessment2$ curl http://app.example.com/data
[
  {
    "age": 30,
    "name": "John Doe"
  },
  {
    "age": 24,
    "name": "Lucky"
  },
  {
    "age": 45,
    "name": "abav"
  }
]


POST Method
-----------
3. curl -X POST http://app.example.com/data -H "Content-Type: application/json" -d '{"name": "lakshman", "age": 24}'

   In this post method, we can post the values to the backend, which will save the data to the mongodb server 
   we make use of Service Headless to route the data to the mongodb and save it

ubuntu@ip-172-31-2-172:~/assessment2$ curl -X POST http://app.example.com/data -H "Content-Type: application/json" -d '{"name": "lakshman", "age": 24}'
{
  "status": "Data inserted"
}

Troubleshoot
-----------
Pod Not Starting: Check pod logs and events using 'kubectl logs <pod-name>' and 'kubectl describe pod <pod-name>'.
Service Not Accessible: Ensure Ingress is configured correctly and DNS is pointing to the right IP.
Authentication errors during browsing: get inside the mongodb pod using kubectl exec -it <mongodbpodname> -- bash
	run "mongosh --host <db-service-name> --username root --password password --authenticationDatabase admin" inside the pod to get verify username and password
	inside mongosh " show dbs "  to view database created.
		




Cleanup Resources
-----------------
kubectl delete -f ingress.yml
kubectl delete -f pv.yml
kubectl delete -f db-headless.yml
kubectl delete -f db-statefulset.yml
kubectl delete -f webapp-service.yml
kubectl delete -f webapp-deployment.yml

