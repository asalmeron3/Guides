# Setting Up Cloud Microservices

## Remote Config Repository
  You will need to have a repository that will contain property files for each of your applications. Each properties file contains the server port that your application will run on and any properties values that your app will have for using within the app.


1. Create your a remote repository (in this case a github repo) with a README and a Java .gitignore
2. Keep track of the link to get to your repository (it will be needed in your config server)

  * In this config repository, we will create .properties files for each app that we want to use the config server in order to pass the properties. 


## Config Server
  You then need to create and run a config server. This will be used so that applications built can ask the config server on what port to run and what properties to include in itself.


  1. Build you project using SpringBoot Initializr. The dependency you need is:  Config Server
  2. In your config project, go to the main application and add the annotation "@EnableConfigServer"
  3. In your config project, go to application.properties and add the information it needs to connect your remote config repo.
  * The port your config server will run on:
      <br> server.port= [customPortNumberHere]

  * The link to how to get to your config repository:
    <br> spring.cloud.config.server.git.uri= [gitHubConfigRepoUrlHere]
  
  #### Example:
    server.port = 8888
    spring.cloud.config.server.git.uri = https://github.com/[githubUsername]/[nameOfConfigRepo]


## Service Registry
  You then need a service registry so that when your build your apps, they will register to the service registry and thus be accessible by other apps if needed.


  1. Build your project using SpringBoot Initializr. The dependency you need is: Eureka Server
  2. In your registry project, go to the main application and add the annotation "@EnableEurekaServer"
  3. In your registry project, go to application.properties and the add the information it needs to build and run
    
  * A port number for your registry to run on. 

    server.port= [portNumberHere]
   
    * Note: A Eureka server's default port number is 8761 and all Eureka clients know to go to that port BY DEFAULT. You can however, run Eureka server on a custom port BUT you will need to explicity tell the clients to go to that custom port.


  * The hostname to use. For apps on your machine it is typically: localhost.
    
    eureka.instance.hostname={hostNameHere}


  * Turn off the client funcionality of the Eureka server (used for High-Availability). This is the only Eureka registry THUS is does not neet to go register with another Eureka server.
    
    eureka.client.registerWithEureka=false <br>
    eureka.client.fetchRegistry=false

  * This is url used by your Eureka clients to register. 
    
    eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka

  ### Example with Eureka's default port (8761)
    server.port = 8761
    eureka.instance.hostname=localhost
    eureka.client.registerWithEureka=false
    eureka.client.fetchRegistry=false
    eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka

  ### Example with custom port number
    server.port = 1234
    eureka.instance.hostname=localhost
    eureka.client.registerWithEureka=false
    eureka.client.fetchRegistry=false
    eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka


## Build Your Apps
   
  1. Build your project using SpringBoot Initializr. The dependencies you need: Spring Web Starter, Spring Boot Actuator, Config Client, & Eureka Discovery Client.
  2. In your current project, go to the main application and add the annotation "@EnableDiscoveryClient" which will allow your project to register to the Eureka Registry and in turn be discovered by other apps
  3. In your app's src/main/resources directory, make a file called bootstrap.properties
  4. In bootstrap properties, provide the information needed to connect to the config server in order for your app to know it's configuration (port number and properties)
  
  * The name of your config file in github; this will also be the name your app uses to register on the registry.
  
    spring.application.name=[nameOfThisAppsPropertiesFileFromGitHubHere]


  * The location of your config server that your set up at the beginning. Follows the pattern http://{hostname}:{portNumber}
  
    spring.cloud.config.uri=[locationOfConfigServer]

  * If your service registry will be running on a custom port (our example above uses port 1234), then you will need this additional line of code: 

    eureka.client.serviceUrl.defaultZone: http://localhost:[customPortNumber]/eureka


  ### Example (IF the registry service is using the default port 8761)
    spring.application.name=moto-inventory-service
    spring.cloud.config.uri=http://localhost:8888

  ### Example (IF the registry service is using a CUSTOM port number)
    spring.application.name=moto-inventory-service
    spring.cloud.config.uri=http://localhost:8888
    eureka.client.serviceUrl.defaultZone: http://localhost:1234/eureka

  5. At this point, we don't have a config file for our app. To make this file, go to gitHub to make a file with the name given above in step 4: [nameOfThisAppsPropertiesFileFromGitHubHere].properties . Following our example, the name of the file would be moto-inventory-service.properties.
  
  In this file, you will have : 

  #### A. Recommended
  * A port to run the moto-inventory-service. If not specified, app will run on 8080.
    
    server.port=[portNumber]


  * This line of code to allow RefreshScope in our app. Refreshscope refreshes the app's configuration without restarting the app when new properties are added to this config file.
    
    management.endpoints.web.exposure.include=*


  #### B. Optional
  * Additional information to include in your properties files will be values for the app to use for itself. This properties COULD be used to find other apps in the registry.


    * General properties/variables
      - greeting=Hello! Welcome to my application!
      - myName=Jon Snow
      - doIUnderstandThis=false
      - whatIKnow=nothing

    * These are additional properties that we will need for THIS service to compose the link/route to the another service (IF NEEDED)
      - theNameOfTheOtherService=[nameHere]
      - theServiceProtocolInThisTutorial=http://
      - theServicePathWeAreUsing=[pathFromOtherServicesControllerToUse]

  ### Example for app NOT using another service
    server.port=7543
    management.endpoints.web.exposure.include=*

    greeting=Hello! Welcome to the Vin Look UP Application!
    vinMaxLength=20
    vinMinLength=5

  ### Example for app that IS using another service
    server.port=7543
    management.endpoints.web.exposure.include=*

    greeting = Welcome to the Moto Inventory App! This app relies a service in the Vin Lookup App.
    genericMake=Toyota
    defaultModel=Corolla

    theNameOfTheOtherService=vin-lookup
    theServiceProtocolInThisTutorial=http://
    theServicePathWeAreUsing=/vin

  6. If your application needs a controller, add the annotation "@RefreshScope" to your controller

  7. If your application needs to communicate with another app (that is registered), make sure your controller has the following information:


  #### Required:
  
  ```Java
  @Autowired
  private DiscoveryClient client;


  private RestTemplate template = new RestTemplate();
  ```

  * DiscoveryClient: Will be used to go to the registry and find/return all instances of the name_of_the_service passed in as a parameter
  * RestTemplate: Will be used to make our request to the other service and return that service's response


  #### Optional (Getting values from the .properties file):
  
  ```Java
  @Value("${propertyNameFromConfigFile}")
    private String nameForThisProperty;
  ```

  * Note: Getting the values from the .properties file (the config file) is optional. You can bring in all the values if your are going to use them.

  8. In your app's controller, whichever endpoint needs to use a microservice's api, compose the uri/link to that endpoint:

  ```Java
  List<ServiceInstance> instances = client.getInstances(microserviceName);
  ServiceInstance microservice = instances.get(0);

  String microserviceApiPath = serviceProtocal + microservice.getHost()
          + ":" + microservice.getPort() + serviceEndpoint;

  // The response can be a String, HashMap<>, Object, etc. depending on what the microservice is returning
  String response = template.getForObject(microserviceApiPath, String.class);
  ```

  ### Example using properties from config file:

  ```Java
  @RestController
  @RefreshScope
  public class MotoInventoryController {

    @Autowired
    private DiscoveryClient client;

    private RestTemplate template = new RestTemplate();

    @Value("${theNameOfTheOtherService}")
    private String otherServiceName;
    @Value("${theServiceProtocolInThisTutorial}")
    private String serviceProtocol;
    @Value("${theServicePathWeAreUsing}")
    private String servicePath;

    @RequestMapping(value = "/vehicle/{vinNumber}", method = RequestMethod.GET)
    @ResponseStatus(HttpStatus.OK)
    public String getVehicleByVin(@PathVariable String vinNumber) {
       // Make a list of all the instances that DiscoveryClient finds with the given name
        List<ServiceInstance> instances = client.getInstances(otherServiceName);
        
        // Get the first instance of the services found
        ServiceInstance otherService = instances.get(0);

        // Compose the uri for the endpoint of the other service
        String otherServiceUri = protocol + otherService.getHost() + ":" + otherService.getPort() + servicePath + "/" + vinNumber;

        // Use RestTemplate to send a request to the other service and store the response
        String responseFromOtherServiceAsAString = template.getForObject(otherServiceUri, HashMap.class);

        return responseFromOtherServiceAsAString;
    }
  }
  ```

  ### Example NOT using properties from config file and just knowing your other service's endpoint:

  ```Java
  @RestController
  @RefreshScope
  public class MotoInventoryController {

    @Autowired
    private DiscoveryClient client;

    private RestTemplate template = new RestTemplate();

    @RequestMapping(value = "/vehicle/{vinNumber}", method = RequestMethod.GET)
    @ResponseStatus(HttpStatus.OK)
    public String getVehicleByVin(@PathVariable String vinNumber) {
        // Make a list of all the instances that DiscoveryClient finds with the given name
        List<ServiceInstance> instances = client.getInstances("vin-lookup");
        
        // Get the first instance of the services found
        ServiceInstance otherService = instances.get(0);

        // Compose the uri for the endpoint of the other service from just knowing the endpoint
        String otherServiceUri = "http://" + otherService.getHost() + ":" + otherService.getPort() + "/vin/" + vinNumber;

        // Use RestTemplate to send a request to the other service and store the response
        String responseFromOtherServiceAsAString = template.getForObject(otherServiceUri, HashMap.class);

        return responseFromOtherServiceAsAString;
    }
  }
  ```
