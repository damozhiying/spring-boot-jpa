# Overview

This project is a revisit an older one that I did a few years back when
I took part in the [Coursera](https://www.coursera.org/) Specialization
on [Android Development](https://www.coursera.org/specializations/android-app-development).

During the final Capstone we were given the choice of several projects
to implement and I chose one that was both interesting and personal: an
application for cancer patients to self-report on their pain symptoms.

A brief article about this specific project was published on the
Vanderbilt School of Engineering's [web site](http://engineering.vanderbilt.edu/news/2014/capstone-app-project-for-mooc-aims-to-track-help-manage-cancer-patients-pain/).

## History

The original server-side implementation of the Capstone project was my
first time using Spring Boot and since then I've felt that there were
many ways to improve upon the original.
  
Since the final Capstone project was only a few weeks long, I did not
get as much time to dedicate to the server-side as I would have liked. I
still had to implement a complete Android front-end for the application
as well (something I had never done).

## Goals

I'd like to re-design the original server-side implementation of the
Capstone project and take this opportunity to document the process so I
can use it as a reference application to share with other developers.

# Step 1: "Initializ" the application

As with most Spring Boot apps, start with the [Spring Initializr](http://start.spring.io/)
web site.  In the Dependencies section, type in: `Web, Actuator, JPA, H2, and Devtools`
and click the Generate button (feel free to customize the Group and Artifact
Id as you see fit).

Unzip the downloaded artifact and import the project into the IDE of
your preference.

Next, "Run" the application.  Depending on your IDE, this might be a
right-click command on the Application class that was automatically
generated from the Initializr.

If all goes well, you should see several INFO commands printed out to
the Console and a Tomcat instance being started on port 8080.
  
Try it out now at: (http://localhost:8080).

You should see the default "Whitelablel" error page.  

Don't worry, this is the Spring Boot default error page that is
indicating that you've requested a resource that doesn't exist.  It
doesn't exist because we haven't implemented anything yet.

Side Note: If you use the `curl` command instead of the browser, you'll
get a JSON response instead as Spring Boot is attempting to detect the
origin of the caller and return the most appropriate response type.
Ultimately, we'll create a custom error handler for these scenarios but
for now lets jump into some actual coding.

# Step 2: Start with the Domain

Lets start by creating a domain object.  By domain object, I'm referring
to an object that represents the business entity we're trying to model.
For more information on Domain Modeling, refer to Eric Evans' book on
[Domain Driven Design](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215).

In Java we can use the standard persistence API (called JPA) to
implement this domain object.  These are basic Java classes that have a
special `@Entity` annotation on them.

To create an entity, create a `domain` package under the default
application package.  Then, create a Java class called `Patient` and add
the following code:

```java
@Entity
public class Patient {
    @Id
    @GeneratedValue
    private Long id;
}
```

Restart the application in the IDE.

If you watch the logs closely, you'll see some additional information
being printed out from a library called Hibernate (this is Spring Boot's
default JPA implementation).

What you may not have noticed however is that by including H2 as a
dependency you are also running an in-memory database with a full web
console:
http://localhost:8080/h2-console

FYI: Make sure you set the JDBC URL to 'jdbc:h2:mem:testdb' instead of 
the default or else you won't see the PATIENT table that was create when
we started the app (this is the Spring Boot default database URL).

Sweet.  This is pretty nice but how do we get the application to be able
to create, read, update and delete Patients?

# Step 3: Create a CRUD repository

Create a package called `repositories` in the base package and create a
`PatientRepository` Java interface class.
  
Finally, add the following code:

```
public interface PatientRepository extends JpaRepository<Patient, Long> {
}
```

That's not a lot of code but what does it do?

By extending the JpaRepository class we creating a Spring "Repository"
class that is automatically initialized with a JPA entity manager.  Not
only is it automatically initialized, it also has several basic
CRUD-type operations already pre-defined.

TODO: talk about why I'm not using @RestRepositories

Before we get too far ahead of ourselves, lets enable some basic
configuration options that will help with debugging.  Edit the default
`application.properties` file in the src/main/resources folder and add:

```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

I *STRONGLY* recommend adding these properties and keeping them set for
the duration of your local development lifecycle.  You will always want
to review the SQL that is being created and executed on your behalf.

# Step 4: Set up a REST Controller

Create a package called `controllers` in the default package and create
a `PatientController` class within it.  Then add the following code to
the class: 

```java
@RestController
public class PatientController {

    private final PatientRepository patientRepository;

    @Autowired
    public PatientController(PatientRepository patientRepository) {
        this.patientRepository = patientRepository;
    }
    
    @GetMapping("/patient/{id}")
    public Patient getPatient(@PathVariable("id") Long id) {
        return patientRepository.findOne(id);
    }
}
```

Restart the application.

After the application restarted you should now be able to make "RESTful"
calls against the /patient URL like this: (http://localhost:8080/patient/1)

However, you will notice that nothing is displayed... that's weird.

TODO discuss findOne behavior of returning null

# Step 5: Create a FindBy Implementation

Since the default behavior really isn't all that desirable, lets add a
more appropriate method to the Repository interface:

```java
public interface PatientRepository extends JpaRepository<Patient, Long> {
    Optional<Patient> findById(Long id);
}
```

and change the GET implementation in the controller to:

```java
    @GetMapping("/patient/{id}")
    public Patient getPatient(@PathVariable("id") Long id) {
        return patientRepository.findById(id).orElseThrow(() -> new IllegalArgumentException("Not found"));
    }
```

Apply these changes and restart.

Now if we use curl:

```
curl -sS localhost:8080/patient/1 | jq
```

we will at least get a 500 status error code.  However, that is not the
most desirable response from a REST API.

TODO Discuss Spring's lack of standard http error exceptions and handlers

# Step 6

Create a package called `exceptions` and add a Java class called `NotFoundException`
with the following code:

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class NotFoundException extends RuntimeException {
    public NotFoundException() {
        super();
    }
    
    public NotFoundException(String message) {
        super(message);
    }
}
```

and modify the GET implementation to throw this instead of the IllegalArgumentException:

```java
        return patientRepository.findById(id)
                .orElseThrow(() -> new NotFoundException(String.format("Patient %d not found", id)));
```

Restart the application and attempt the `curl` command again.

```json
{
  "timestamp": 1481488691203,
  "status": 404,
  "error": "Not Found",
  "exception": "com.undertree.symptom.exceptions.NotFoundException",
  "message": "Patient 1 not found",
  "path": "/patient/1"
}
```

That looks better!

# Step 7:  Add a Patient

We have a functional GET but now we need a way to add a Patient.  Using
REST semantics this should be expressed with a POST.  To support this,
add the following code to the PatientController:

```java
    @PostMapping("/patient")
    public Patient addPatient(@RequestBody Patient patient) {
        return patientRepository.save(patient);
    }
```

Restart and issue a `curl -H "Content-Type: application/json" -X POST localhost:8080/patient -d '{}'`

Interestingly we get a "{}" back.  If we look at the logs we should be
able to see that an insert did take place but where is the id of the
entity we just created?

If we open up the H2 console again, we should see that a row was created
with the ID of 1.  If we use our GET operation, we should get the entity
back right?

```
curl -sS localhost:8080/patient/1 | jq
```

Still the same empty empty object?

The reason we don't see the id field is that we failed to provide a getter
method on the entity class.  The default Jackson library that Spring uses
to convert the object to JSON needs getters and setters to be able to
access object's internal values.

Modify the Patient class to add a getter for the id field:

```java
    public Long getId() {
        return id;
    }
```

Now when you execute the POST we can see an id field on the JSON object

```bash
curl -sS -H "Content-Type: application/json" -X POST localhost:8080/patient -d '{}' | jq
```

TODO discuss Lombok

# Step 8:  Lets make this Patient more interesting

Add the following attributes to the Patient entity:

```java
    private String givenName;
    private String familyName;
    private LocalDate birthDate;
```

Add the associated getter/setters using your IDEs code generator if
possible.

Using curl again submit a Patient add request:

```bash
curl -sS -H "Content-Type: application/json" -X POST localhost:8080/patient -d '{"givenName":"Max","familyName":"Colorado","birthDate":"1942-12-11"}' | jq
```

400 Error?  So that LocalDate is causes a JsonMappingException.

We need to add a Jackson dependency to the project so it knows how to
handle JSR 310 Dates (introduced in Java 8).  Add this to the pom.xml:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
    <version>2.8.5</version>
</dependency>

```

Refresh the dependencies and restart the application.  Now we see an odd
looking date structure:

```json
"birthDate": [
    1942,
    12,
    11
  ]
```

That isn't exactly what we want.  There is yet another configuration
change we need here to tell Jackson to format the date "correctly".

Update the application.properties and include:

```
spring.jackson.serialization.WRITE_DATES_AS_TIMESTAMPS = false
```

Volia!

But wait.  If we look at how the date is actually stored in the database
through the H2 console, it looks like a BLOB.  That isn't good since
we might want to be able to query against it.

Why doesn't JPA support LocalDate?

# Step 9: Create a JPA Converter

As of the time of writing, JPA still does not natively support the JSR
310 dates.  It does, however, provide support for custom converters that
can.

Add a package called `converters` and add a class called `LocalDateAttributeConverter`.
 Then add the following code:
 
```java
@Converter(autoApply = true)
public class LocalDateAttributeConverter implements AttributeConverter<LocalDate, Date> {

    @Override
    public Date convertToDatabaseColumn(LocalDate localDate) {
        return (localDate == null ? null : Date.valueOf(localDate));
    }

    @Override
    public LocalDate convertToEntityAttribute(Date sqlDate) {
        return (sqlDate == null ? null : sqlDate.toLocalDate());
    }
}
```

When you restart the application you might see in the logs that the
table is now being create with a proper DATE type like this:

```
Hibernate: 
    create table patient (
        id bigint generated by default as identity,
        birth_date date,
        family_name varchar(255),
        given_name varchar(255),
        primary key (id)
    )
```

That is a huge improvement!  If you use other Java 8 Date types, you'll
need a converter for each (I can't believe no one has created a utility
library for these yet).

# Step 10:  Let's write some tests

So far we've not written a lot of code but its still a good idea to get
into the habit of writing unit tests.  Spring Boot 1.4 provides some new
testing capabilities that we want to take advantage of.

First, lets create a test for our PatientRepository.  In the test section,
create a package called `repositories` and then add a Java class called
`PatientRepositoryTests`.  In that class add the following code:

```java
@RunWith(SpringRunner.class)
@DataJpaTest
public class PatientRepositoryTests {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    PatientRepository patientRepository;

    @Test
    public void test_PatientRepository_FindById_ExpectExists() throws Exception {
        Long patientId = entityManager.persistAndGetId(new Patient(), Long.class);
        Patient aPatient = patientRepository.findById(patientId).orElseThrow(NotFoundException::new);
        assertThat(aPatient.getId()).isEqualTo(patientId);
    }
}
```

Run the test and review the logs.  You should see several SQL statements
being executed by the JPA provider.  In this case we see the PATIENT table
CREATE, INSERT, SELECT and finally DROP.

Specifically with the INSERT and SELECT statements, is this similar to
what you might have written by hand?

Spoiler Alert!  I expect that this test case will fail at some point in
the future.  Can you guess why?

Now let's add a test for the Controller.  Create a package called
`controllers` and create a test class called `PatientControllerTests`.
Then add the following code:

```java
@RunWith(SpringRunner.class)
@WebMvcTest(PatientController.class)
public class PatientControllerTests {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private PatientRepository mockPatientRepository;

    @Test
    public void test_MockPatient_Expect_ThatGuy() throws Exception {
        given(mockPatientRepository.findById(1L)).willReturn(Optional.of(thatGuy()));

        mockMvc.perform(get("/patient/1")
                .accept(APPLICATION_JSON_UTF8))
                .andExpect(status().isOk())
                .andExpect(content().contentType(APPLICATION_JSON_UTF8))
                .andExpect(jsonPath("$.givenName", is("Guy")))
                .andExpect(jsonPath("$.familyName", is("Stromboli")))
                .andExpect(jsonPath("$.birthDate", is("1942-11-21")));
    }

    private Patient thatGuy() {
        Patient aPatient = new Patient();
        aPatient.setGivenName("Guy");
        aPatient.setFamilyName("Stromboli");
        aPatient.setBirthDate(LocalDate.of(1942, 11, 21));
        return aPatient;
    }
}
```

This is a "mock" test.  It is not testing the repository that we created
but instead is leveraging a mock version of it thanks to Mockito.

Basically, all we are doing is telling the mock how to behave when we
call it and then are asserting that it is responding correctly.  It is a
good test but it isn't really a complete test end-to-end test.  The
overall benefit of this kind of test is when you want to test certain
behaviors that might be hard to recreate under normal circumstances.

Next, lets add a more complete web test.  In the same package, add a
test class called `PatientControllerWebTests`.  Then add the following
code:

```
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment=RANDOM_PORT)
public class PatientControllerWebTests {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    public void test_PatientController_getPatient_Expect_Patient1_Exists() {
        ResponseEntity<Patient> entity = restTemplate.getForEntity("/patient/1", Patient.class);

        assertThat(entity.getStatusCode()).isEqualTo(HttpStatus.OK);

        Patient aPatient = entity.getBody();
        assertThat(aPatient).isNotNull();
        assertThat(aPatient.getGivenName()).isEqualTo("Phillip");
        assertThat(aPatient.getFamilyName()).isEqualTo("Spec");
        assertThat(aPatient.getBirthDate()).isEqualTo(LocalDate.of(1972, 5, 5));
    }
```

If you run this test right now, it will fail.  Why?  Well, the obvious
answer is that when we run the test the database is initialized as an
empty data set.  How do we initialize the database so we can rely on
a specific data set?

One way is to tap into a feature of Spring Boot.  If we create a file
in the test section of the `resources` folder called `data.sql` that
file will be used to initialize the database on ever test case.

Create that file and add the following:

```
INSERT INTO PATIENT(id, given_name, family_name, birth_date) VALUES (null, 'Phillip', 'Spec', '1972-5-5');
INSERT INTO PATIENT(id, given_name, family_name, birth_date) VALUES (null, 'Sally', 'Certify', '1973-6-6');
```

During the startup of the tests, Spring Boot will invoke this file and
initialize the database.

Now, if we run the test cases, they should succeed with a green bar!
Yeah!

# Step 11:  What about performance?

TODO setup a performance test.

# Next

## TODOs
- Add @Valid
- Add example mocking tests
- Add performance tests
- Add QueryDsl support
- Add custom response wrapping
- Add REST documentation Swagger vs RESTDocs
- Add Spring Security with OAuth2 and JWT
- Explain HIPA and PII concerns


# Additional Resources

- [Spring Boot Reference Guide]:(https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Data JPA]:(http://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- 