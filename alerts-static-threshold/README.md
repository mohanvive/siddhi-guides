Generating Alerts Based on Static and Dynamic Thresholds
===========================================================

In this guide, you are going to understand one of the common requirements of a Stream Processing which is generating 
alerts based on static and dynamic thresholds. To understand this requirement, let’s consider the throttling use case 
in an API management solutions. 

# Scenario - Throttling for API Requests
Throttling became as one of the unavoidable needs with the evolution of  APIs and API management. Throttling is a 
process that is used to control the usage of APIs by consumers during a given period. 

The following sections are available in this guide.

* [What you'll build](#what-youll-build)
* [Prerequisites](#prerequisites)
* [Implementation](#implementation)
* [Testing](#testing)
* [Deployment & Output](#deployment)

## What you'll build

Let's consider a real world use case to implement the throttling requirement. This will help you to 
understand some Siddhi Stream Processing constructs such as windows, aggregations, source and etc… Let’s jump in 
to the use case directly.

Let's assume that you are an API developer and you have published a few APIs to the API store. There are also 
subscribers who have subscribed to these APIs. These APIs are subscribed by the users in different tier and there 
tiers are categorized based on number of requests per min/sec. If any of the subscriber is consuming the API more 
than the allowed quota with in a time frame then that specific user is throttled until that time frame passes.

For example, let’s assume that user “John” has subscribed to an API with the tier 'Silver'; silver tier allows a user 
to make 10 API requests per minute. If John, made more than 10 requests within a minute then his subsequent requests 
get throttled.

![Throttling_Scenario](images/throttling-scenario.png "Throttling Scenario")

Now, let’s understand how this could be implemented in Siddhi engine.


## Prerequisites
Below are the prerequisites that should be considered to implement above use case.

### Mandatory Requirements
* Siddhi tooling distribution
* Siddhi runner distribution 
    - [VM/Local Runtime](https://github.com/siddhi-io/distribution/releases/download/v5.1.0-m1/siddhi-runner-5.1.0-m1.zip )
    - [Docker Image](https://hub.docker.com/r/siddhiio/siddhi-tooling)
    - K8S Operator (commands are given in deployment section)
* Java 8 or higher

### Requirements needed to deploy in Docker/Kubernetes

* [Docker](https://docs.docker.com/engine/installation/)
* [Minikube](https://github.com/kubernetes/minikube#installation) or [Google Kubernetes Engine(GKE) Cluster](https://console.cloud.google.com/) or [Docker for Mac](https://docs.docker.com/docker-for-mac/install/)


## Implementation

When a subscriber made an API call to `order-mgt-v1` API it sends an event to Siddhi runtime through HTTP transport. 

* Siddhi runtime, keep track of each API request and make decisions to throttle subscribers. 
* Again, once required time interval passed Siddhi release those throttle users. 
* Throttling decisions are informed  to API management solution through an API call.
* If a subscriber is getting throttled more than 10 times in an hour then sends a notification mail to that user requesting to upgrade the tier.


### Implement Streaming Queries

1. Start the Siddhi [tolling(https://siddhi.io/en/v5.0/docs/tooling/) runtime and go to the editor UI in http://localhost:9390/editor 

2. Select File -> New option, then you could either use the source view or design view to write/build the Siddhi Application. Below is the Siddhi Application, implemented to cater the requirements mentioned above.

3. Let’s write (develop) the Siddhi  Application, as given below.

4. Once above Siddhi app is created, you can use the Event Simulator option in the editor to Simulate events to streams and perform developer testing.


````
@App:name('API-Request-Throttler')
@App:description('Enforcesthrottling to API requests ')

-- HTTP endpoint which listens for api request related events
@source(type = 'http', receiver.url = "http://0.0.0.0:8006/apiRequestStream", basic.auth.enabled = "false", 
	@map(type = 'json'))
define stream APIRequestStream (apiName string, version string, tier string, user string, userEmail string);

-- HTTP sink to publich throttle decisions. For testing purpose, there is a mock logger service provided 
@sink(type = 'http', publisher.url = "http://${LOGGER_SERVICE_HOST}:8080/logger", method = "POST", 
	@map(type = 'json'))
@sink(type = 'log', @map(type = 'text'))
define stream ThrottleOutputStream (apiName string, version string, user string, tier string, userEmail string, isThrottled bool);

-- Email sink to send alerts
@sink(type = 'log', @map(type = 'text'))
@sink(type = 'email', username = "${EMAIL_USERNAME}", address = "${SENDER_EMAIL_ADDRESS}", password = "${EMAIL_PASSWORD}", 
subject = "Upgrade API Subscription Tier", to = "{{userEmail}}", host = "smtp.gmail.com", port = "465", ssl.enable = "true", auth = "true", 
	@map(type = 'text', 
		@payload("""
			Hi {{user}}

			You have subscribed to API called {{apiName}}:{{version}} with {{tier}} tier.
			Based on our records, it seems you are hitting the upper limit of the API requests in a frequent manner. 

			We kindly request you to consider upgrading to next API subscription tier to avoid this in the future. 

			Thanks, 
			API Team""")))
define stream UserNotificationStream (user string, apiName string, version string, tier string, userEmail string, throttledCount long);

@info(name = 'Query to find users who needs to be throttled based on tier `silver`')
from APIRequestStream[tier == "silver"]#window.timeBatch(1 min, 0, true) 
select apiName, version, user, tier, userEmail, count() as totalRequestCount 
	group by apiName, version, user 
	having totalRequestCount > 10 or totalRequestCount == 0 
insert all events into ThrottledStream;

@info(name = 'Query to find users who needs to be throttled based on tier `gold`')
from APIRequestStream[tier == "gold"]#window.timeBatch(1 min, 0, true) 
select apiName, version, user, tier, userEmail, count() as totalRequestCount 
	group by apiName, version, user 
	having totalRequestCount > 100 or totalRequestCount == 0 
insert all events into ThrottledStream;

@info(name = 'Query to add a flag for throttled request')
from ThrottledStream 
select apiName, version, user, tier, userEmail, ifThenElse(totalRequestCount == 0, false, true) as isThrottled 
insert into ThrottleOutputStream;

@info(name = 'Query to find frequently throttled users - who have throttled more than 10 times in the last hour')
from ThrottleOutputStream[isThrottled]#window.time(1 hour) 
select user, apiName, version, tier, userEmail, count() as throttledCount 
	group by user, apiName, version, tier 
	having throttledCount > 10 
output first every 15 min 
insert into UserNotificationStream;

````

Source view of the Siddhi app.
![source_view](images/source-view.png "Source view of the Siddhi App")

Below is the flow diagram of the above Siddhi App.

![query_flow](images/query-flow.png "Query flow diagram")


## Testing

NOTE: In the above provided Siddhi app, there are some environmental variables (EMAIL_PASSWORD, EMAIL_USERNAME, 
SENDER_EMAIL_ADDRESS, and LOGGER_SERVICE_HOST)  are used. These values are required to be set to send an email alert 
based on the Siddhi queries defined.

- You could simply simulate some events directly into the stream and test your Siddhi app in 
the editor itself. Once server is started, you will see below logs get printed in the editor console.

![editor_app_run_mode](images/editor-app-run-mode.png "Siddhi App Run mode")

Then, you simulate some events through HTTP to test the behavior. Below is the cURL request that you can use to test the behavior. 

### Run Mock Logger service

In the provided Siddhi app, there is a HTTP sink configured to push output events to an HTTP
endpoint. To verify that, please download the mock server 
[jar](https://github.com/mohanvive/siddhi-mock-services/releases/download/v1.0.0/logservice-1.0.0.jar) and run that 
mock service by executing below command.

````
java -jar logservice-1.0.0.jar
````

### Invoking the Service

As mentioned in the previous steps, there is a service running in Siddhi side which is listening
for events related to API requests. As per the Siddhi query that you wrote in the
‘Implementation’ section, respective service can be accessed via
`http://localhost:9090/ThotttleService` .
As per the app, the API request will get throttled if there is more than 10 requests by the same
user, to the same API (for 'silver’ tier).

````
curl -v -X POST -d '{ "event": { "apiName": "order-mgt-v1", "version": "1.0.0", "tier":"silver","user":"mohan", "userEmail":"mohan@wso2.com"}}' "http://localhost:8006/apiRequestStream" -H "Content-Type:application/json"
````

If you invoke, above cURL request for more than 10 times within a minute then Siddhi start
throttling the request and send an alert to a service in the API Manager. In this guide, for the
simplicity purpose you can just log the alert as below.

````
INFO {io.siddhi.core.stream.output.sink.LogSink} - API-Request-Throttler :
ThrottleOutputStream : Event{timestamp=1564056341280, data=[order-mgt-v1, 1.0.0,
mohan, silver, true], isExpired=false}
````

At the same time, you can see the Throttled related events get logged in the mock service
console.

![simulation_mock_service_output](images/simulation-mock-service-output.png "Simulation output @ mock service")

If a user gets throttled more than 10 times within an hour then Siddhi sends an email to the
respective user. Note: please make sure to change the config values of the email sink in the
Siddhi query provided in above.

![simulation_email_output](images/simulation-email-output.png "Generated email for th simulation")

## Deployment

Once you are done with the development, export the Siddhi app that you have developed with File -> Export File option.

You can deploy the Siddhi app using any of the methods listed below. 

Note: In the above provided Siddhi app, there are some environmental variables (EMAIL_PASSWORD, EMAIL_USERNAME, 
and SENDER_EMAIL_ADDRESS)  are used. These values are required to be set to send an email alert based on the Siddhi 
queries defined. Again, there is a mock service configured to publish the throttle decisions (given below instructions). 
Please configure LOGGER_SERVICE_HOST environment property to point the host where mock service is running.

### Deploy on VM/ Bare Metal

1. Download the latest Siddhi Runner [distribution](https://github.com/siddhi-io/distribution/releases/download/v0.1.0/siddhi-runner-0.1.0.zip)
2. Unzip the siddhi-runner-x.x.x.zip
3. Start SiddhiApps with the runner config by executing the following commands from the distribution directory
        
     ```
     Linux/Mac : ./bin/runner.sh -Dapps=<siddhi-file-path> 
     Windows : bin\runner.bat -Dapps=<siddhi-file-path>

	    Eg: If exported siddhi app in Siddhi home directory,
            ./bin/runner.sh -Dapps=API-Request-Throttler.siddhi
      ```
        
4. Download the mock [logging service](https://github.com/mohanvive/siddhi-mock-services/releases/download/v1.0.0/logservice-1.0.0.jar) which is used to demonstrate the capability of SIddhi HTTP sink. Execute below command to run the mock server.

	    java -jar logservice-1.0.0.jar

5. Invoke the service with below cURL request for more than 10 times within a minute time period.

        curl -v -X POST -d '{ "event": { "apiName": "order-mgt-v1", "version": "1.0.0", "tier": "silver", "user":"mohan", "userEmail":"mohan@wso2.com"}}' "http://localhost:8006/apiRequestStream" -H "Content-Type:application/json"

6. You can see the output log in the console as shown below. You could see there is an alert log printed as shown in the below image.

     ![console_output_vm](images/console-output-vm.png "Console output")

7. At the same time, you could see the events received to HTTP mock endpoint that started in the step [3].

      ![mock_service_output_vm](images/mock-service-output-vm.png "Mock service output")

###Deploy on Docker

1. Create a folder locally (eg: /home/siddhi-apps) and copy the Siddhi app in to it.

2. Pull the the latest Siddhi Runner image from [Siddhiio Docker Hub] (https://hub.docker.com/u/siddhiio).
    
    ````
    docker pull siddhiio/siddhi-runner-alpine:5.1.0-m1
    ````

3. Start SiddhiApp by executing the following docker command.

    ````
    docker run -it -p 8006:8006 -v /home/siddhi-apps:/apps -e EMAIL_PASSWORD=siddhi123 -e EMAIL_USERNAME=siddhi.gke.user -e SENDER_EMAIL_ADDRESS=siddhi.gke.user@gmail.com -e LOGGER_SERVICE_HOST=10.100.0.99 siddhiio/siddhi-runner-alpine:5.1.0-m1 -Dapps=/apps/API-Request-Throttler.siddhi
    ````

    NOTE:In the above provided Siddhi app, there are some environmental variables (EMAIL_PASSWORD, EMAIL_USERNAME, 
    and SENDER_EMAIL_ADDRESS)  are used. These values are required to be set to send an email alert based on the 
    Siddhi queries defined. Again, there is a mock service configured to publish the throttle decisions (given below instructions). 
    Please configure LOGGER_SERVICE_HOST environment property to point the host where mock service is running. 
    Hence, make sure to add proper values for the environmental variables in the above command.

4. Download the mock [logging service](https://github.com/mohanvive/siddhi-mock-services/releases/download/v1.0.0/logservice-1.0.0.jar) which is used to demonstrate the capability of SIddhi HTTP sink. Execute below command to run the mock server.

    ````
	    java -jar logservice-1.0.0.jar
	````

5. Invoke the service with below cURL request for more than 10 times within a minute time period.

    ````
        curl -v -X POST -d '{ "event": { "apiName": "order-mgt-v1", "version": "1.0.0", "tier": "silver", "user":"mohan", "userEmail":"mohan@wso2.com"}}' "http://localhost:8006/apiRequestStream" -H "Content-Type:application/json"
    ````
        
6. Since, you have started the docker in interactive mode you can see the output in the console as below. 
(If it is not started in the interactive mode then you can use `docker exec -it  <docker-container-id> sh` command, 
go in to the container and check the log file in `home/siddhi_user/siddhi-runner/wso2/runner/logs/carbon.log` file)

    ![console_output_docker](images/console-output-docker.png "Console output")

7. At the same time, you could see the events received to HTTP mock endpoint that started in the step [4].

    ![mock_service_output_docker](images/mock-service-output-docker.png "Mock service output")


### Deploying on Kubernetes
1. Install Siddhi Operator
    - To install the Siddhi Kubernetes operator run the following commands.
        
        ````
        kubectl apply -f https://github.com/siddhi-io/siddhi-operator/releases/download/v0.2.0-m1/00-prereqs.yaml
        kubectl apply -f https://github.com/siddhi-io/siddhi-operator/releases/download/v0.2.0-m1/01-siddhi-operator.yaml
        ````
        
     - You can verify the installation by making sure the following deployments are running in your Kubernetes cluster.
     
        ![kubernetes_siddhi-deployments](images/kubernetes-siddhi-deployments.png "K8S Siddhi deployments")


2. Download the mock [logging service](https://github.com/mohanvive/siddhi-mock-services/releases/download/v1.0.0/logservice-1.0.0.jar) 
which is used to demonstrate the capability of SIddhi HTTP sink. Execute below command to run the mock server.

    ````
        java -jar logservice-1.0.0.jar
    ````
    
3. Siddhi applications can be deployed on Kubernetes using the Siddhi operator.
    - You have to define an [Ingress](https://kubernetes.github.io/ingress-nginx/deploy/#provider-specific-steps) since 
    there is an http endpoint in the Siddhi app which you will be sending events to that

    - To deploy the above created Siddhi app, you have to create custom resource object yaml file (with the kind as SiddhiProcess) as given below
    
        ````
        apiVersion: siddhi.io/v1alpha2
        kind: SiddhiProcess
        metadata:
          name: api-throttler-app
        spec:
          apps:
           - script: |
                @App:name('API-Request-Throttler')
                @App:description('Enforcesthrottling to API requests ')
        
                -- HTTP endpoint which listens for api request related events
                @source(type = 'http', receiver.url = "http://0.0.0.0:8006/apiRequestStream", basic.auth.enabled = "false", 
                    @map(type = 'json'))
                define stream APIRequestStream (apiName string, version string, tier string, user string, userEmail string);
        
                -- HTTP sink to publich throttle decisions. For testing purpose, there is a mock logger service provided 
                @sink(type = 'http', publisher.url = "http://${LOGGER_SERVICE_HOST}:8080/logger", method = "POST", 
                    @map(type = 'json'))
                @sink(type = 'log', @map(type = 'text'))
                define stream ThrottleOutputStream (apiName string, version string, user string, tier string, userEmail string, isThrottled bool);
        
                -- Email sink to send alerts
                @sink(type = 'log', @map(type = 'text'))
                @sink(type = 'email', username = "${EMAIL_USERNAME}", address = "${SENDER_EMAIL_ADDRESS}", password = "${EMAIL_PASSWORD}", 
                subject = "Upgrade API Subscription Tier", to = "{{userEmail}}", host = "smtp.gmail.com", port = "465", ssl.enable = "true", auth = "true", 
                    @map(type = 'text', 
                        @payload("""
                            Hi {{user}}
        
                            You have subscribed to API called {{apiName}}:{{version}} with {{tier}} tier.
                            Based on our records, it seems you are hitting the upper limit of the API requests in a frequent manner. 
        
                            We kindly request you to consider upgrading to next API subscription tier to avoid this in the future. 
        
                            Thanks, 
                            API Team""")))
                define stream UserNotificationStream (user string, apiName string, version string, tier string, userEmail string, throttledCount long);
        
                @info(name = 'Query to find users who needs to be throttled based on tier `silver`')
                from APIRequestStream[tier == "silver"]#window.timeBatch(1 min, 0, true) 
                select apiName, version, user, tier, userEmail, count() as totalRequestCount 
                    group by apiName, version, user 
                    having totalRequestCount > 10 or totalRequestCount == 0 
                insert all events into ThrottledStream;
        
                @info(name = 'Query to find users who needs to be throttled based on tier `gold`')
                from APIRequestStream[tier == "gold"]#window.timeBatch(1 min, 0, true) 
                select apiName, version, user, tier, userEmail, count() as totalRequestCount 
                    group by apiName, version, user 
                    having totalRequestCount > 100 or totalRequestCount == 0 
                insert all events into ThrottledStream;
        
                @info(name = 'Query to add a flag for throttled request')
                from ThrottledStream 
                select apiName, version, user, tier, userEmail, ifThenElse(totalRequestCount == 0, false, true) as isThrottled 
                insert into ThrottleOutputStream;
        
                @info(name = 'Query to find frequently throttled users - who have throttled more than 10 times in the last hour')
                from ThrottleOutputStream[isThrottled]#window.time(1 hour) 
                select user, apiName, version, tier, userEmail, count() as throttledCount 
                    group by user, apiName, version, tier 
                    having throttledCount > 10 
                output first every 15 min 
                insert into UserNotificationStream;
        
          container:
            env:
              -
                name: EMAIL_PASSWORD
                value: "siddhi123"
              -
                name: EMAIL_USERNAME
                value: "siddhi.gke.user"
              - 
                name: SENDER_EMAIL_ADDRESS
                value: "siddhi.gke.user@gmail.com"
              -
                name: LOGGER_SERVICE_HOST
                value: "10.100.0.99"
        
            image: "siddhiio/siddhi-runner-ubuntu:5.1.0-m1"
        ````
        
        NOTE: In the above provided Siddhi app, there are some environmental variables (EMAIL_PASSWORD, EMAIL_USERNAME, 
        and SENDER_EMAIL_ADDRESS)  are used. These values are required to be set to send an email alert based on the 
        Siddhi queries defined. Again, there is a mock service configured to publish the throttle decisions (given below instructions). 
        Please configure LOGGER_SERVICE_HOST environment property to point the host where mock service is running. 
        Hence, make sure to add proper values for the environmental variables in the above yaml file (check the `env` section of the yaml file).
        
    - Now,  let’s create the above resource in the Kubernetes  cluster with below command.
        
        ````    	
    	 kubectl create -f <absolute-yaml-file-path>/API-Request-Throttler.yaml
    	````
    
    - Once, Siddhi app is successfully deployed. You can verify its health with below Kubernetes commands
        
        ![kubernetes_output](images/kubernetes-output.png "Kubectl console outputs")
        
    - Then, add the host siddhi and related external IP (ADDRESS) to the /etc/hosts file in your
    machine. In this case, external IP is `0.0.0.0`.
    
    - You can find the alert logs in the siddhi runner log file. To see the Siddhi runner log file, you can invoke below command.
        
        ````
        kubectl get pods
        ````
    
        Then, find the pod name of the siddhi app deployed. Then invoke below command,
        
        ````
        kubectl logs <siddhi-app-pod-name> -f
        ````
        
        Eg: as shown below image,
            
         ![kubernetes_get_pods](images/kubernetes-get-pods.png "`Kubectl get pods` command output ")

    - Invoke the Throttle event receiver service with below cURL request for more than 10 times within a minute.
      
        ````
        curl -v -X POST -d '{ "event": { "apiName": "order-mgt-v1", "version": "1.0.0", "tier": "silver", "user":"mohan", "userEmail":"mohan@wso2.com"}}' "http://siddhi/api-throttler-app-0/8006/apiRequestStream" -H "Content-Type:application/json"
        ````
          
    - Then, you could see the throttle decisions as console logs (as given below).
    
        ![console_output_kubernetes](images/console-output-kubernetes.png "Console output")
    
    - At the same time, you could see the events received to HTTP mock endpoint that started in the step [2].
    
        ![mock_service_output_kubernetes](images/mock-service-output-kubernetes.png "Mock service output")
    
    
   NOTE: Refer more details in https://siddhi.io/en/v5.1/docs/siddhi-as-a-kubernetes-microservice/ 

