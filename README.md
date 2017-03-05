# spring-boot-rest-tutorial
Spring Boot Rest Tutorial
http://briansjavablog.blogspot.in/2015/12/spring-boot-rest-tutorial.html
Spring Boot REST Tutorial
Spring Boot makes it easier to build Spring based applications by focusing on convention over configuration.  Following standard Spring Boot conventions we can minimize the configuration required to get an application up and running. The use of an embedded servlet container allows us to package the application as an executable JAR and simply invoke it on the command line to launch the application.
One of my favorite things about Boot is its emphasis on production readiness. Out of the box it provides a number of key non functional features, such as metrics, health checks and externalized configuration. In the past these types of features would have been written from scratch (or more worrying, not at all), before an application could be considered production ready.

This tutorial is an introduction to Spring Boot and describes the steps required to build and test a simple JPA backed REST service. The code is intended to be simple, easy to follow and provide readers with a template for building more elaborate services.

Source Code  
The full source code for this tutorial is available on github at https://github.com/briansjavablog/spring-boot-rest-tutorial. You may find it useful to have the code locally so that you can experiment with it as you work through the tutorial. 

Main Application Class
We'll start off by looking at the main Application class. The first thing thing you'll probably notice is a main method that calls SpringApplication.run. This is used to launch the application and apply configuration based on the annotation values specified at the top of the class.

package com.blog.samples;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}

The presence of a main method in a web application may seem a little strange at first, but this allows us to run the application as a simple executable JAR. No longer do we need to build a WAR and deploy it to a servlet container. Instead we can simply execute the JAR, which will launch and run the application in an embedded servlet container. By default Boot uses Tomcat but it's possible to use Jetty instead if that's your preference. For the the sake of this tutorial we'll stick with Tomcat.

Now lets look at the the class annotations that provide the base configuration for our application.
@SpringBootApplication - a wrapper annotation that automatically includes the following common configuration annotations
@Configuration - registers the class as a source of beans for Springs Application Context. 
@EnableAutoConfiguration - Boot uses this to configure the application based on JAR dependencies we've added to the POM.
@ComponentScan - Tells Spring to look for and register beans from base package com.blog.samples and all of its sub packages.
Domain Model
Before we can create our REST endpoints we need to define a domain model. To keeps things really simple we'll have just 2 entities, a Customer and their Address.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
package com.blog.samples.boot.rest.model;

import java.util.Date;

import javax.persistence.CascadeType;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.OneToOne;

import lombok.Getter;
import lombok.Setter;

@Entity
public class Customer{

    public Customer(){}
 
    public Customer(String firstName, String lastName, Date dateOfBirth, Address address) {
        super();
        this.firstName = firstName;
        this.lastName = lastName;
        this.dateOfBirth = dateOfBirth;
        this.address = address;
    }


    @Id
    @Getter
    @GeneratedValue(strategy=GenerationType.AUTO)
    private long id;
 
    @Setter
    @Getter
    private String firstName;
 
    @Setter
    @Getter
    private String lastName;
 
    @Setter 
    @Getter
    private Date dateOfBirth;

    @Setter
    @Getter
    @OneToOne(cascade = {CascadeType.ALL})
    private Address address;
}

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
package com.blog.samples.boot.rest.model;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

import lombok.Getter;
import lombok.Setter;

@Entity
public class Address{

    public Address(){}
 
    public Address(String street, String town, String county, String postcode) {
        this.street = street;
        this.town = town;
        this.county = county;
        this.postcode = postcode;
    }

    @Id
    @Getter
    @GeneratedValue(strategy=GenerationType.AUTO)
    private long id;
 
    @Setter
    @Getter
    private String street;
 
    @Setter
    @Getter
    private String town;
 
    @Setter   
    @Getter
    private String county;

    @Setter
    @Getter
    private String postcode;
}

Our sample application will use a a simple hsql in memory database to store customer data. We'll create a simple JPA repository later, but first need to configure the domain objects so that they can be mapped to database tables.
@Entity - registers the bean as a JPA entity. By default this entity is mapped to a database table with the same name. If we wanted to map this entity to a database table with a different name we would specify the table name with @Entity(name="app_customer")
@Id - marks the id instance variable as the primary key field.
@GeneratedValue(strategy=GenerationType.AUTO) - indicates that primary keys will be generated automatically when a new instance is persisted. As a result, the application is not responsible for setting the entity Id before persisting the object.
@OneToOne(cascade = {CascadeType.ALL}) - maps a one to one relationship between Customer and Address. CascadeType.ALL allows us to create a new Customer with a new Address, and persist both with a single save.The same cascading action applies when deleting a Customer (associated Address is also deleted)  
@Setter/@Getter - these are nothing to do with JPA, just convenient lombock annotations for generating getters and setters.

JPA Repository
Next we'll create a JPA repository for persisting the entities created above. Creating a repository couldn't be simpler as Spring Data does all the heavy lifting for us. We simply create an interface that extends CrudRepository and supply our Customer target type. We don't need to create an implementation as Spring Data will use the information supplied to route requests to the appropriate JPA CRUD repository implementation our our behalf. As a result we get a fully functional CRUD repository with a bunch of common data access methods implemented for us.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
package com.blog.samples.boot.rest.repository;

import java.util.List;

import org.springframework.data.repository.CrudRepository;
import org.springframework.stereotype.Repository;

import com.blog.samples.boot.rest.model.Customer;


public interface CustomerRepository extends CrudRepository<Customer, Long> {

    public List<Customer> findByFirstName(String firstName); 
}

I've added a custom query called findByFirstName. Spring Data recognizes that firstName is an instance variable on Customer and so provides us with an implementation for this query.

REST Controller
Now that most of the supporting components are in place, we can start looking at the REST endpoints. The controller we're going to build will expose the following CRUD endpoints.

HTTP Action
URL
Purpose
GET
http://localhost:8080/rest/customers/1
Returns JSON representation of Customer resource based on Id specified in URL (in this case 1)
GET
http://localhost:8080/rest/customers
Returns JSON representation of all available Customer resources 
POST
http://localhost:8080/rest/customers
Create a new Customer resource using JSON representation supplied in the HTTP request body. The path to the newly created Customer resource is returned in a HTTP header as Location: /rest/customers/6 
PUT
http://localhost:8080/rest/customers/1
Update existing Customer resource with JSON representation supplied in the HTTP request body. 
DELETE
http://localhost:8080/rest/customers/4
Delete the Customer specified by the Id in the URL.


The first step is to create a Controller class and annotate it with @RestController.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
package com.blog.samples.boot.rest.controller;

import java.util.List;

import javax.servlet.http.HttpServletResponse;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.context.request.WebRequest;

import com.blog.samples.boot.rest.exception.CustomerNotFoundException;
import com.blog.samples.boot.rest.exception.InvalidCustomerRequestException;
import com.blog.samples.boot.rest.model.Customer;
import com.blog.samples.boot.rest.repository.CustomerRepository;

/**
 * Customer Controller exposes a series of RESTful endpoints
 */
@RestController
public class CustomerController {

    @Autowired
    private CustomerRepository customerRepository;

 
@RestController is a convenience annotation that registers the class as a controller for handling incoming HTTP requests, similar to @Controller. It has the added benefit of automatically applying @ResponseBody to each of the endpoint methods that return an entity, @ResponseBody is responsible for handling the serialization of the response object so that its JSON representation can be written to the HTTP response body. 
@Autowired CustomerRepostory is the JPA repository we created earlier. We'll use it later to retrieve and persist Customer entities,

Get Customer Endpoint
Below is the endpoint definition for handling a HTTP Get request for a specific Customer resource.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
    @RequestMapping(value = "/rest/customers/{customerId}", method = RequestMethod.GET)
    public Customer getcustomer(@PathVariable("customerId") Long customerId) {
  
        /* validate customer Id parameter */
        if (null==customerId) {
            throw new InvalidCustomerRequestException();
        }
  
        Customer customer = customerRepository.findOne(customerId);
  
        if(null==customer){
            throw new CustomerNotFoundException();
        }
  
        return customer;
    }
@RequestMapping annotation supplies 2 important pieces of information. The value attribute defines the URL pattern supported and the method attribute defines the HTTP method supported. Springs dispatcher servlet inspects incoming HTTP requests and uses both pieces of configuration to route the appropriate requests to this endpoint.   
@PathVariable annotation strips customer Id parameter from request URL and maps it to customer Id method parameter,
Lines 5&6 check if a customer Id has been supplied and if not throws a custom InvalidCustomerRequestException. Later we'll create a custom exception handler to convert this application exception to an appropriate HTTP response code.
Line 9 uses the CustomerRepository to retrieve the specified Customer entity from the database,
Lines 11 & 12 check if the requested Customer was found. If not we throw a custom CustomerNotFoundException. 

Endpoint Exception Handling
In the endpoint method above we came across 2 custom exceptions. Given that this is a RESTful endpoint, its important we return the appropriate HTTP response code in the event of an error. Spring provides a neat way of mapping application exceptions to HTTP response codes, by allowing us to define a custom exception handler outside of the controller.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
package com.blog.samples.boot.rest.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;

import lombok.extern.slf4j.Slf4j;

@Slf4j
@ControllerAdvice
public class ControllerExceptionHandler {

    @ResponseStatus(HttpStatus.NOT_FOUND) // 404
    @ExceptionHandler(CustomerNotFoundException.class)
    public void handleNotFound() {
        log.error("Resource not found");
    }

    @ResponseStatus(HttpStatus.BAD_REQUEST) // 400
    @ExceptionHandler(InvalidCustomerRequestException.class)
    public void handleBadRequest() {
        log.error("Invalid Fund Request");
    }
 
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR) // 500
    @ExceptionHandler(Exception.class)
    public void handleGeneralError(Exception ex) {
        log.error("An error occurred procesing request", ex);
    }
}

This class handles all application exceptions thrown by the controller endpoints. Each method corresponds to a particular type of exception and contains 2 key pieces of configuration
@ExceptionHandler - defines the application exception that this method handles. We can define any exception type that extends Throwable. Spring calls the handleNotFound method when CustomerNotFoundException is thrown by an endpoint.
@ResponseStatus - defines the HTTP response code returned to the client when this type of exception is thrown. This allows us to map any kind of application exception to the most suitable HTTP response code. The handleNotFound method has been configured so that a HTTP 404 is returned when CustomerNotFoundException is thrown by an endpoint. This provides the client application with the correct semantic context when the requested resource is not available.  
The thrown exception is passed to the method as an argument so we're free to use it as we please. In this instance I simply log the error, but in a production application we may want to do something more interesting, like gather metrics for different types of failure.

Get All Customers Endpoint
Below is the endpoint definition for handling HTTP Get requests for all Customer resources,

1
2
3
4
5
@RequestMapping(value = "/rest/customers", method = RequestMethod.GET)
public List<Customer> getCustomers() {
  
    return (List<Customer>) customerRepository.findAll();
}

This endpoint method is very simple indeed and uses the CustomerRepository to return all Customer resources from the database, The request mapping configuration is similar to that defined earlier for the Get Customer by Id endpoint. The obvious difference is that the URL template and the method signature do not expect an Id parameter.

Create Customer Endpoint
Now lets take a look at how we create a new Customer resource. The endpoint defined below does exactly that.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
@RequestMapping(value = { "/rest/customers" }, method = { RequestMethod.POST })
public Customer createCustomer(@RequestBody Customer customer, HttpServletResponse httpResponse, WebRequest request) {

    Customer createdcustomer = null;
    createdcustomer = customerRepository.save(customer);  
    httpResponse.setStatus(HttpStatus.CREATED.value());
    httpResponse.setHeader("Location", String.format("%s/rest/customers/%s", request.getContextPath(), customer.getId()));
 
    return createdcustomer;
}
@RequestMapping - contains the URL pattern that this endpoint will handle requests for. The method attribute is POST, indicating that this method will process incoming HTTP POST requests. 
@RequestBody - Requests are expected to contain a JSON representation of a Customer resource in the request body. We use the @RequestBody annotation on the method signature to map the incoming JSON payload to a Customer POJO.
Line 5 - save the Customer entity to the database using JPA repository we created earlier.
Line 6 - Set a HTTP Created 201 code on the response so that the client knows the resource was created successfully.
Line 7 - Set a Location header on the HTTP response with a URL providing the client with a handle on the resource they've just create
Line 9 - return an updated representation of the resource that's just been created. 

Update Customer Endpoint
Next we'll create an update endpoint so that clients can apply updates to an existing Customer resource.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
@RequestMapping(value = { "/rest/customers/{customerId}" }, method = { RequestMethod.PUT })
public void updateCustomer(@RequestBody Customer customer, @PathVariable("customerId") Long customerId, HttpServletResponse httpResponse) {

    if(!customerRepository.exists(customerId)){
        httpResponse.setStatus(HttpStatus.NOT_FOUND.value());
    }
    else{
        customerRepository.save(customer);
        httpResponse.setStatus(HttpStatus.NO_CONTENT.value()); 
    }
}
@RequestMapping - contains the URL pattern that this endpoint will handle requests for. The method attribute is PUT, indicating that this method will process incoming HTTP PUT requests. 
@RequestBody - Requests are expected to contain a JSON representation of a Customer resource in the request body. We use the @RequestBody annotation on the method signature to map the incoming JSON payload to a Customer POJO.
Lines 4 & 5 - check that the supplied entity already exists and if it doesn't, set a HTTP 404 response code.
Lines 8 & 9 - Save the supplied entity. JPA will use the Customer Id to update the existing entity.Finally we set a HTTP 204 No Content to let the client know the operation succeeded but that we haven't returned anything in the response body.

Delete Customer Endpoint
The final endpoint we're going to create is for deleting an existing entity.

1
2
3
4
5
6
7
8
9
@RequestMapping(value = "/rest/customers/{customerId}", method = RequestMethod.DELETE)
public void removeCustomer(@PathVariable("customerId") Long customerId, HttpServletResponse httpResponse) {

    if(customerRepository.exists(customerId)){
        customerRepository.delete(customerId); 
    }
  
    httpResponse.setStatus(HttpStatus.NO_CONTENT.value());
}
@RequestMapping - contains the URL pattern that this endpoint will handle requests for. The method attribute is DELETE, indicating that this method will process incoming HTTP DELETE requests. 
@PathVariable annotation strips customer Id parameter from request the URL and maps it to customer Id method parameter,
Lines 4 & 5 - check that the supplied entity already exists and if it does, use the JPA repository to delete it.
Lines 8 -  Finally we set a HTTP 204 No Content to let the client know the operation succeeded, but that we haven't returned anything in the response body.

Test Data
We've now created all 5 endpoints, but before we can run the service we need to setup some test data. We'll use 2 classes, one to supply the test data and another to load that data on application startup. I've deliberately split this into two classes as I want to reuse the data provider later as part of the integration tests. The data provider simply returns a list of Customer objects as shown below.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
package com.blog.samples.boot.rest.data;

import java.util.Arrays;
import java.util.List;

import org.joda.time.DateTime;
import org.springframework.stereotype.Component;

import com.blog.samples.boot.rest.model.Address;
import com.blog.samples.boot.rest.model.Customer;

@Component
public class DataBuilder {
 
    public List<Customer> createCustomers() {

        Customer customer1 = new Customer("Joe", "Smith", DateTime.parse("1982-01-10").toDate(),
             new Address("High Street", "Belfast", "Down", "BT893PY"));

        Customer customer2 = new Customer("Paul", "Jones", DateTime.parse("1973-01-03").toDate(),
             new Address("Main Street", "Lurgan", "Armagh", "BT283FG"));

        Customer customer3 = new Customer("Steve", "Toale", DateTime.parse("1979-03-08").toDate(),
             new Address("Main Street", "Newry", "Down", "BT359JK"));
  
        return Arrays.asList(customer1, customer2, customer3);
    }
}

Next we'll create a DataLoader and configure it to run on application startup.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
package com.blog.samples.boot.rest.data;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.stereotype.Component;

import com.blog.samples.boot.rest.repository.CustomerRepository;

import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
public class DataLoader implements ApplicationListener<ContextRefreshedEvent>{

    @Autowired
    private DataBuilder dataBuilder;
 
    @Autowired
    private CustomerRepository customerRepository;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent contextRefreshedEvent) {

        log.debug("Loading test data...");
        dataBuilder.createCustomers().forEach(customer -> customerRepository.save(customer));
        log.debug("Test data loaded...");
    }
}

On application startup Spring calls the onApplicationEvent method to get data from the DataProvider and save it to the database. 


Endpoint Integration Tests
Now that we have everything in place its time to write some integration tests to prove it all works. I like integration tests for rest endpoints as they allow us to test the HTTP endpoint, controller logic and and data access as a single unit. If this were production code we would of course supplement the integration tests with a suit of fine grained unit tests using mocked dependencies. For the sake of this tutorial though, we'll stick with the integration tests.

 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = Application.class)
@WebAppConfiguration
@IntegrationTest({"server.port=0"})
public class CustomerControllerIT {

 @Value("${local.server.port}")
 private int port;
 private URL base;
 private RestTemplate template;

 @Autowired
 private DataBuilder dataBuilder;
 
 @Autowired
 private CustomerRepository customerRepository;
 
 private static final String JSON_CONTENT_TYPE = "application/json;charset=UTF-8"; 
 

In the class header we use a number of annotations to configure the tests. We need to tell Spring that this is an integration test and that it requires a WebApplicationContext to run.
A RestTemplate and URL are used to build the HTTP requests we'll send to the endpoints. We've also injected the DataBuilder which we'll use below to setup test data prior to each test. 

Next we'll add a setup method that will run before each test, to build the base URL and set up test data.

1
2
3
4
5
6
7
8
9
@Before
public void setUp() throws Exception {
    this.base = new URL("http://localhost:" + port + "/rest/customers");
    template = new TestRestTemplate();  
  
    /* remove and reload test data */
    customerRepository.deleteAll();  
    dataBuilder.createCustomers().forEach(customer -> customerRepository.save(customer));  
}

Get All Customers Test
1
2
3
4
5
6
7
8
@Test
public void getAllCustomers() throws Exception {
    ResponseEntity<String> response = template.getForEntity(base.toString(), String.class);  
    assertThat(response.getStatusCode(), equalTo(HttpStatus.OK));
  
    List<Customer> customers = convertJsonToCustomers(response.getBody());  
    assertThat(customers.size(), equalTo(3));  
}

This test sends a HTTP Get to localhost:8080/rest/customers and verifies that HTTP response code is 200. We use a convenience method to convert the JSON payload in the response body to a list of Customer objects, and check that 3 objects have been returned. 

Get Customer By Id Test
 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
@Test
public void getCustomerById() throws Exception {
  
    Long customerId = getCustomerIdByFirstName("Joe");
    ResponseEntity<String> response = template.getForEntity(String.format("%s/%s", base.toString(), customerId), String.class);
    assertThat(response.getStatusCode(), equalTo(HttpStatus.OK));
    assertThat(response.getHeaders().getContentType().toString(), equalTo(JSON_CONTENT_TYPE));
  
    Customer customer = convertJsonToCustomer(response.getBody());
  
    assertThat(customer.getFirstName(), equalTo("Joe"));
    assertThat(customer.getLastName(), equalTo("Smith"));
    assertThat(customer.getDateOfBirth().toString(), equalTo("Sun Jan 10 00:00:00 GMT 1982"));
    assertThat(customer.getAddress().getStreet(), equalTo("High Street"));
    assertThat(customer.getAddress().getTown(), equalTo("Belfast"));
    assertThat(customer.getAddress().getCounty(), equalTo("Down"));
    assertThat(customer.getAddress().getPostcode(), equalTo("BT893PY"));
}

We start by looking up the target customer Id using the JPA repository created earlier. We then send a HTTP Get to localhost:8080/rest/customers/{ID} and verify the HTTP response code and content type. We convert the JSON payload in the response body to a a Customer object and check that the object is populated as expected.

Create Customer Test
 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
@Test
public void createCustomer() throws Exception {

    Customer customer = new Customer("Gary", "Steale", DateTime.parse("1984-03-08").toDate(),
       new Address("Main Street", "Portadown", "Armagh", "BT359JK"));

    ResponseEntity<String> response = template.postForEntity("http://localhost:" + port + "/rest/customers", customer, String.class);
    assertThat(response.getStatusCode(), equalTo(HttpStatus.CREATED));
    assertThat(response.getHeaders().getContentType().toString(), equalTo(JSON_CONTENT_TYPE));
    assertThat(response.getHeaders().getFirst("Location"), containsString("/rest/customers/"));
  
    Customer returnedCustomer = convertJsonToCustomer(response.getBody());  
    assertThat(customer.getFirstName(), equalTo(returnedCustomer.getFirstName()));
    assertThat(customer.getLastName(), equalTo(returnedCustomer.getLastName()));
    assertThat(customer.getDateOfBirth(), equalTo(returnedCustomer.getDateOfBirth()));
    assertThat(customer.getAddress().getStreet(), equalTo(returnedCustomer.getAddress().getStreet()));
    assertThat(customer.getAddress().getTown(), equalTo(returnedCustomer.getAddress().getTown()));
    assertThat(customer.getAddress().getCounty(), equalTo(returnedCustomer.getAddress().getCounty()));
    assertThat(customer.getAddress().getPostcode(), equalTo(returnedCustomer.getAddress().getPostcode()));
}

In this test we create a Customer object, convert it to JSON and do a HTTP POST to localhost:8080/rest/customers. We verify the response type, content type, the existence of a Location header and the response body to ensure it contains a JSON representation of a Customer entity. The JSON is converted to an object for verification.

Update Customer Test
 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
@Test
public void updateCustomer() throws Exception {

    Long customerId = getCustomerIdByFirstName("Joe");
    ResponseEntity<String> getCustomerResponse = template.getForEntity(String.format("%s/%s", base.toString(), customerId), String.class);
    assertThat(getCustomerResponse.getStatusCode(), equalTo(HttpStatus.OK));
    assertThat(getCustomerResponse.getHeaders().getContentType().toString(), equalTo(JSON_CONTENT_TYPE));
  
    Customer returnedCustomer = convertJsonToCustomer(getCustomerResponse.getBody());
    assertThat(returnedCustomer.getFirstName(), equalTo("Joe"));
    assertThat(returnedCustomer.getLastName(), equalTo("Smith"));
    assertThat(returnedCustomer.getDateOfBirth().toString(), equalTo("Sun Jan 10 00:00:00 GMT 1982"));
    assertThat(returnedCustomer.getAddress().getStreet(), equalTo("High Street"));
    assertThat(returnedCustomer.getAddress().getTown(), equalTo("Belfast"));
    assertThat(returnedCustomer.getAddress().getCounty(), equalTo("Down"));
    assertThat(returnedCustomer.getAddress().getPostcode(), equalTo("BT893PY"));
  
    /* convert JSON response to Java and update name */
    ObjectMapper mapper = new ObjectMapper();
    Customer customerToUpdate = mapper.readValue(getCustomerResponse.getBody(), Customer.class);
    customerToUpdate.setFirstName("Wayne");
    customerToUpdate.setLastName("Rooney");

    /* PUT updated customer */
    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_JSON); 
    HttpEntity<Customer> entity = new HttpEntity<Customer>(customerToUpdate, headers); 
    ResponseEntity<String> response = template.exchange(String.format("%s/%s", base.toString(), customerId), HttpMethod.PUT, entity, String.class, customerId);
  
    assertThat(response.getBody(), nullValue());
    assertThat(response.getStatusCode(), equalTo(HttpStatus.NO_CONTENT));

    /* GET updated customer and ensure name is updated as expected */
    ResponseEntity<String> getUpdatedCustomerResponse = template.getForEntity(String.format("%s/%s", base.toString(), customerId), String.class);
    assertThat(getCustomerResponse.getStatusCode(), equalTo(HttpStatus.OK));  
    assertThat(getCustomerResponse.getHeaders().getContentType().toString(), equalTo(JSON_CONTENT_TYPE));
  
    Customer updatedCustomer = convertJsonToCustomer(getUpdatedCustomerResponse.getBody());
    assertThat(updatedCustomer.getFirstName(), equalTo("Wayne"));
    assertThat(updatedCustomer.getLastName(), equalTo("Rooney"));
    assertThat(updatedCustomer.getDateOfBirth().toString(), equalTo("Sun Jan 10 00:00:00 GMT 1982"));
    assertThat(updatedCustomer.getAddress().getStreet(), equalTo("High Street"));
    assertThat(updatedCustomer.getAddress().getTown(), equalTo("Belfast"));
    assertThat(updatedCustomer.getAddress().getCounty(), equalTo("Down"));
    assertThat(updatedCustomer.getAddress().getPostcode(), equalTo("BT893PY"));
}

This test has a number of steps. We begin by issuing a HTTP GET to retrieve an existing Customer resource. We then update that entity and issue a HTTP PUT request with the updated Customer passed as JSON in the request body. Finally we issue another HTTP GET request to retrieve the updated representation of the Customer resource and verify that it has been updated as expected.

Delete Customer Test
 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
@Test
public void deleteCustomer() throws Exception {

    Long customerId = getCustomerIdByFirstName("Joe");  
    ResponseEntity<String> response = template.getForEntity(String.format("%s/%s", base.toString(), customerId), String.class);
    assertThat(response.getStatusCode(), equalTo(HttpStatus.OK));
    assertThat(response.getHeaders().getContentType().toString(), equalTo(JSON_CONTENT_TYPE));
  
    Customer customer = convertJsonToCustomer(response.getBody());
    assertThat(customer.getFirstName(), equalTo("Joe"));
    assertThat(customer.getLastName(), equalTo("Smith"));
    assertThat(customer.getDateOfBirth().toString(), equalTo("Sun Jan 10 00:00:00 GMT 1982"));
    assertThat(customer.getAddress().getStreet(), equalTo("High Street"));
    assertThat(customer.getAddress().getTown(), equalTo("Belfast"));
    assertThat(customer.getAddress().getCounty(), equalTo("Down"));
    assertThat(customer.getAddress().getPostcode(), equalTo("BT893PY"));
  
    /* delete customer */
    template.delete(String.format("%s/%s", base.toString(), customerId), String.class);
  
    /* attempt to get customer and ensure qwe get a 404 */
    ResponseEntity<String> secondCallResponse = template.getForEntity(String.format("%s/%s", base.toString(), customerId), String.class);
    assertThat(secondCallResponse.getStatusCode(), equalTo(HttpStatus.NOT_FOUND));
}

This test begins by retrieving an existing Customer resource using a HTTP GET request. We then issue a HTTP DELETE request to remove the entity, followed by another HTTP GET to check that the entity is no longer available.

Running from the Command Line
The integration tests we created above are handy but its also useful to test the service from the command line with CURL. To run the sample code from the command line follow the instructions below.
cd to spring-boot-rest directory
run mvn clean  package
run java -jar target/spring-boot-rest-0.1.0.jar and the application should start up as shown below.


Testing from the Command Line
Below are a few sample CURL commands for calling the service from the command line. This is nice handy way of sending HTTP requests to an endpoint without the hassle of creating a client application.

Get Customer by Id
curl -i localhost:8080/rest/customers/1











Get All Customers
curl -i localhost:8080/rest/customers








Create New Customer (POST)
curl -i -H "Content-Type: application/json" -X POST -d '{"firstName":"JoeXXXXXXXXXXXXXXX","lastName":"SmithXXXXXXXXXXXXX","dateOfBirth":379468800000,"address":{"street":"High Street","town":"Belfast","county":"Down","postcode":"BT893PY"}}' localhost:8080/rest/customers








Update Customer (PUT)
 curl -i -H "Content-Type: application/json" -X PUT -d '{"id":3,"firstName":"Joe","lastName":"Smith333333","dateOfBirth":379468800000,"address":{"id":3,"street":"High Street","town":"Belfast","county":"Down","postcode":"BT893PY"}}' localhost:8080/rest/customers/3











Delete Customer
curl -i -X DELETE localhost:8080/rest/customers/2









Summary
After reading this post you should have the knowledge required to build and test a simple RESTful service with Spring Boot. The full source code for this tutorial is available on Github at  
https://github.com/briansjavablog/spring-boot-rest-tutorial, so feel free to download it and have a play around. If you found this material useful, please share it with others or leave a comment below.
Posted by Brian 
