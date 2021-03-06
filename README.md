# Spring Boot JPA Reference Application

[![Build Status](https://travis-ci.org/ssherwood/spring-boot-jpa.svg?branch=master)](https://travis-ci.org/ssherwood/spring-boot-jpa)

## Overview

This project is a revisit an older one that I did a few years back when I took part in the
[Coursera](https://www.coursera.org/) Specialization on [Android Development](https://www.coursera.org/specializations/android-app-development).

During the final Capstone we were given the choice of several projects to implement and I chose one that was both
interesting and personal: an application to help cancer patients self-report on their pain symptoms so their doctors
can be notified of extended durations of persistent pain or inability to eat.

A brief article about this specific project was published on the Vanderbilt School of Engineering's
[web site](http://engineering.vanderbilt.edu/news/2014/capstone-app-project-for-mooc-aims-to-track-help-manage-cancer-patients-pain/).

## History

The original server-side implementation of the Capstone project was my first time using Spring Boot and since then I've
felt that there were many ways to improve upon the original code.
  
Since the final Capstone project was only a few weeks long, I did not
get as much time to dedicate to the server-side as I would have liked. I
still had to implement a complete Android front-end for the application
as well (something I had never done).

## Goals

I'd like to re-design the original server-side implementation of the
Capstone project and take this opportunity to document the process so I
can use it as a reference application to share with other developers.

---

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
  
Try it out now at: http://localhost:8080

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
console at http://localhost:8080/h2-console

FYI: Make sure you set the JDBC URL to 'jdbc:h2:mem:testdb' instead of 
the default or else you won't see the PATIENT table that was create when
we started the app (this is the Spring Boot default database URL).

Sweet.  This is pretty nice but how do we get the application to be able
to create, read, update and delete Patients?

# Step 3: Create a JPA Repository

Create a package called `repositories` in the base package and create a
`PatientRepository` Java interface class.
  
Finally, add the following code:

```java
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
    
    @GetMapping("/patients/{id}")
    public Patient getPatient(@PathVariable("id") Long id) {
        return patientRepository.findOne(id);
    }
}
```

Restart the application.

After the application restarted you should now be able to make "RESTful"
calls against the /patients URL like this: http://localhost:8080/patients/1

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
    @GetMapping("/patients/{id}")
    public Patient getPatient(@PathVariable("id") Long id) {
        return patientRepository.findById(id).orElseThrow(() -> new IllegalArgumentException("Not found"));
    }
```

Apply these changes and restart.

Now if we use curl again:

```
curl -sS localhost:8080/patients/1 | jq
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
  "path": "/patients/1"
}
```

That looks better!

# Step 7:  Add a Patient via POST

We have a functional GET but now we need a way to add a Patient.  Using
REST semantics this should be expressed with a POST.  To support this,
add the following code to the PatientController:

```java
    @PostMapping("/patients")
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
curl -sS localhost:8080/patients/1 | jq
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
curl -sS -H "Content-Type: application/json" -X POST localhost:8080/patients -d '{}' | jq
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
curl -sS -H "Content-Type: application/json" -X POST localhost:8080/patients -d '{"givenName":"Max","familyName":"Colorado","birthDate":"1942-12-11"}' | jq
```

400 Error?  So that LocalDate is causes a JsonMappingException.

We need to add a Jackson dependency to the project so it knows how to
handle JSR 310 Dates (introduced in Java 8).  Add this to the pom.xml:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
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

That isn't exactly what we want.
  
There is yet another configuration change we need here to tell Jackson
to format the date "correctly".  Update the application.properties and
include:

```properties
spring.jackson.serialization.WRITE_DATES_AS_TIMESTAMPS=false
```

Volia!

But wait.  If we look at how the date is actually stored in the database
through the H2 console, it looks like a BLOB.  That isn't good since
we might want to be able to query against it later.

Why doesn't JPA support LocalDate?

# Step 9: Create a JPA LocalDate Converter

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

```sql
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

# Step 10:  Let's write some unit tests

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

        mockMvc.perform(get("/patients/1")
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

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment=RANDOM_PORT)
public class PatientControllerWebTests {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    public void test_PatientController_getPatient_Expect_Patient1_Exists() throws Exception {
        ResponseEntity<Patient> entity = restTemplate.getForEntity("/patients/1", Patient.class);

        assertThat(entity.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(entity.getBody()).isNotNull()
                .hasFieldOrPropertyWithValue("givenName", "Phillip")
                .hasFieldOrPropertyWithValue("familyName", "Spec")
                .hasFieldOrPropertyWithValue("birthDate", LocalDate.of(1972, 5, 5));
    }

    @Test
    public void test_PatientController_getPatient_Expect_Patient99999999_NotFound() throws Exception {
        ResponseEntity<Patient> entity = restTemplate.getForEntity("/patients/99999999", Patient.class);
        assertThat(entity.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
    }

    @Test
    public void test_PatientController_getPatient_Expect_Invalid() throws Exception {
        String response = restTemplate.getForObject("/patients/foo", String.class);
        assertThat(response).contains("Bad Request");
    }
}
```

If you run this test right now, it will fail.  Why?
  
Well, the obvious answer is that when we ran the test and the database
is initialized with empty data.  How do we initialize the database so we
can rely on a specific data set?

One way is to tap into a feature of Spring Boot.  If we create a file
in the test section of the project called `resources` and then a file
called `data.sql` within it we have a way to initialize the database on
ever test case.

Create that file and add the following to it:

```sql
INSERT INTO PATIENT(id, given_name, family_name, birth_date) VALUES (null, 'Phillip', 'Spec', '1972-5-5');
INSERT INTO PATIENT(id, given_name, family_name, birth_date) VALUES (null, 'Sally', 'Certify', '1973-6-6');
```

During the startup of the tests, Spring Boot will invoke this file and
initialize the database.  Now, if we re-run the test cases, they should
succeed with a green bar!

Yeah!

# Step 11:  What about performance?

TODO setup a performance test.
https://github.com/jmeter-maven-plugin/jmeter-maven-plugin
http://www.xoriant.com/blog/software-testing-and-qa/performance-testing-of-restful-apis-using-jmeter.html

TODO I've add an initial performance plan but this will need to be
better documented as JMeter can be a complex tool in and of itself.
Additionally, I'm still working out details on how to load the in-memory
database using a CSV so I can share the data between the application and
the JMeter test (what I have appears to work but I'm still looking for
better that having to store the CSV in the resources/classpath).

After this, it might be a good idea to go back and refactor this to be
the pattern used from the start.

FYI - The reason I'm wanting to have JMeter set up so early in the life
of the project is so that we can establish base-lines for later as we
add more complex capabilities.  I think its a good idea to have some
sense of what impact to performance a specific feature can have.

Another piece to document would be to connect to the application with
JConsole and then run the test.  Reviewing heap and threads is a good
practice.

Finally, consider introducing the jmeter plugin to the maven POM.

# Step 12: Bean Validation JSR 309/3

There a a few types of validations that we can add to our application
for the entity classes.
  
One type is to use the JPA `@Column` annotations to define constraints
on the resulting database tables.  I typically like to delay this
exercise until the project is a bit more fleshed out.  During this
initial discovery phase, I find it more advantageous to leave the schema
more flexible and adaptable.  At some point in the future I plan on
having the schema be versioned - then we'll need to investigate
additional tooling like Flyway or Liquibase.

Another option for data validation is the Java Bean Validator
annotations:
https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#table-spec-constraints

In the domain object for Patient, lets add a few common sense
validations:

```java
    @NotBlank @Size(min = 2)
    private String givenName;
    @NotBlank @Size(min = 2)
    private String familyName;
```

If you restart the application and look at the Hibernate create table,
you'll see that I was wrong.  Hibernate did honor these validations and
modified the schema accordingly.  Now if I try to post a blank JSON
object to the /patients URL, I should see an error:

```json
{
  "timestamp": "2016-12-14T15:22:39.519+0000",
  "status": 500,
  "error": "Internal Server Error",
  "exception": "javax.validation.ConstraintViolationException",
  "message": "Validation failed for classes [com.undertree.symptom.domain.Patient] during persist time for groups [javax.validation.groups.Default, ]\nList of constraint violations:[\n\tConstraintViolationImpl{interpolatedMessage='may not be null', propertyPath=givenName, rootBeanClass=class com.undertree.symptom.domain.Patient, messageTemplate='{javax.validation.constraints.NotNull.message}'}\n\tConstraintViolationImpl{interpolatedMessage='may not be null', propertyPath=familyName, rootBeanClass=class com.undertree.symptom.domain.Patient, messageTemplate='{javax.validation.constraints.NotNull.message}'}\n]",
  "path": "/patients"
}
```

During persistence I violated one or more constraints and the database
isn't happy.  I'm not happy either because of the 500 error status code.
I'd actually like to catch this condition higher up the stack and return
a more proper error.

Java and Spring have a simple solution for this.  Add a `@Valid`
annotation to the RequestBody parameter like this:

```java
    @PostMapping("/patients")
    public Patient addPatient(@Valid @RequestBody Patient patient) {
        return patientRepository.save(patient);
    }
```

Restart and rerun the POST:

```json
{
  "timestamp": "2016-12-14T15:41:20.115+0000",
  "status": 400,
  "error": "Bad Request",
  "exception": "org.springframework.web.bind.MethodArgumentNotValidException",
  "errors": [
    {
      "codes": [
        "NotNull.patient.givenName",
        "NotNull.givenName",
        "NotNull.java.lang.String",
        "NotNull"
      ],
      "arguments": [
        {
          "codes": [
            "patient.givenName",
            "givenName"
          ],
          "arguments": null,
          "defaultMessage": "givenName",
          "code": "givenName"
        }
      ],
      "defaultMessage": "may not be null",
      "objectName": "patient",
      "field": "givenName",
      "rejectedValue": null,
      "bindingFailure": false,
      "code": "NotNull"
    },
    {
      "codes": [
        "NotNull.patient.familyName",
        "NotNull.familyName",
        "NotNull.java.lang.String",
        "NotNull"
      ],
      "arguments": [
        {
          "codes": [
            "patient.familyName",
            "familyName"
          ],
          "arguments": null,
          "defaultMessage": "familyName",
          "code": "familyName"
        }
      ],
      "defaultMessage": "may not be null",
      "objectName": "patient",
      "field": "familyName",
      "rejectedValue": null,
      "bindingFailure": false,
      "code": "NotNull"
    }
  ],
  "message": "Validation failed for object='patient'. Error count: 2",
  "path": "/patients"
}
```

It 400 error is more in lines with what I was expecting but wow that is
a verbose error.

TODO this would be a good place to look into customizing the Spring Boot
standard error object.  The errors block is just blindly marshalling the
entire object with codes and arguments that aren't all that useful to a
client.  We could probably clean this up a bit.

I did waste some time trying to customized the ValidationMessages and
had a little success but it didn't feel as clean as I would have liked
so I need to do more research there as well.

Lets run or test cases just to be sure everything is working as
expected.  Wait, there was an error:

```
javax.validation.ConstraintViolationException: Validation failed for classes [com.undertree.symptom.domain.Patient] during persist time for groups [javax.validation.groups.Default, ]
List of constraint violations:[
	ConstraintViolationImpl{interpolatedMessage='may not be empty', propertyPath=familyName, rootBeanClass=class com.undertree.symptom.domain.Patient, messageTemplate='{org.hibernate.validator.constraints.NotBlank.message}'}
	ConstraintViolationImpl{interpolatedMessage='may not be empty', propertyPath=givenName, rootBeanClass=class com.undertree.symptom.domain.Patient, messageTemplate='{org.hibernate.validator.constraints.NotBlank.message}'}
]
```

This is what I thought might happen.  Our original test was using an
empty resource during the Add.  Now this is an invalid state.  We should
fix that first.

First add a new library to our dependencies in the pom.xml:

```xml
		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-lang3</artifactId>
			<version>3.5</version>
		</dependency>
```

We'll use this in conjunction with a helpful utility class that can be
added to test suite under a new `domain` package.  This will be a new
Java class called `TestPatientBuilder` with the code:

```java
public class TestPatientBuilder {
    private final Patient testPatient = new Patient();

    public TestPatientBuilder() {
        // Start out with a valid randomized patient
        testPatient.setGivenName(RandomStringUtils.randomAlphabetic(2, 30));
        testPatient.setFamilyName(RandomStringUtils.randomAlphabetic(2, 30));
        LocalDate start = LocalDate.of(1949, Month.JANUARY, 1);
        long days = ChronoUnit.DAYS.between(start, LocalDate.now());
        testPatient.setBirthDate(start.plusDays(RandomUtils.nextLong(0, days + 1)));
    }

    public TestPatientBuilder withGivenName(String givenName) {
        testPatient.setGivenName(givenName);
        return this;
    }

    public TestPatientBuilder withFamilyName(String familyName) {
        testPatient.setFamilyName(familyName);
        return this;
    }

    public TestPatientBuilder withBirthDate(LocalDate birthDate) {
        testPatient.setBirthDate(birthDate);
        return this;
    }

    public Patient build() {
        return testPatient;
    }
}
```

This class lets us create random test patients with the added ability to
override any attribute that we would like.  This can be very useful in
unit testing.

Now, try it out buy updating the failing test case:

```java
   @Test
    public void test_PatientRepository_FindById_ExpectExists() throws Exception {
        Long patientId = entityManager.persistAndGetId(new TestPatientBuilder().build(), Long.class);
        Patient aPatient = patientRepository.findById(patientId).orElseThrow(NotFoundException::new);
        assertThat(aPatient.getId()).isEqualTo(patientId);
    }
```

We should go ahead and add a few negative tests that certify the
validations that we recently added.  First add a JUnit rule for
exceptions:

```java
    @Rule
    public ExpectedException thrown = ExpectedException.none();
```

Now when we expect an exception to be thrown we'll set up the `thrown`
with our expectations:

```java
    @Test
    public void test_PatientRepository_SaveWithNull_ExpectException() throws Exception {
        thrown.expect(InvalidDataAccessApiUsageException.class);
        patientRepository.save((Patient)null);
    }

    @Test
    public void test_PatientRepository_SaveWithEmpty_ExpectException() throws Exception {
        thrown.expect(ConstraintViolationException.class);
        thrown.expectMessage(containsString("'may not be empty'"));
        patientRepository.save(new Patient());
    }

    @Test
    public void test_PatientRepository_SaveWithEmptyGivenName_ExpectException() throws Exception {
        thrown.expect(ConstraintViolationException.class);
        thrown.expectMessage(allOf(containsString("givenName"), containsString("'may not be empty'")));
        patientRepository.save(new TestPatientBuilder().withGivenName("").build());
    }

    @Test
    public void test_PatientRepository_SaveWithEmptyFamilyName_ExpectException() throws Exception {
        thrown.expect(ConstraintViolationException.class);
        thrown.expectMessage(allOf(containsString("familyName"), containsString("'may not be empty'")));
        patientRepository.save(new TestPatientBuilder().withFamilyName("").build());
    }

    @Test
    public void test_PatientRepository_SaveWithShortGivenName_ExpectException() throws Exception {
        thrown.expect(ConstraintViolationException.class);
        thrown.expectMessage(allOf(containsString("givenName"), containsString("'size must be between 2 and")));
        patientRepository.save(new TestPatientBuilder().withGivenName("A").build());
    }

    @Test
    public void test_PatientRepository_SaveWithShortFamilyName_ExpectException() throws Exception {
        thrown.expect(ConstraintViolationException.class);
        thrown.expectMessage(allOf(containsString("familyName"), containsString("'size must be between 2 and")));
        patientRepository.save(new TestPatientBuilder().withFamilyName("Z").build());
    }
```

Not only can we assert that a specific exception is thrown but we can
verify that it contains the message details that we would expect.

Finally, lets refactor and update the Controller tests.  In the
`PatientControllerTests` refactor to use the `TestPatientBuilder`:

```java
        given(mockPatientRepository.findById(1L))
                .willReturn(Optional.of(new TestPatientBuilder()
                        .withGivenName("Guy")
                        .withFamilyName("Stromboli")
                        .withBirthDate(LocalDate.of(1942, 11, 21))
                        .build()));
```

In the `PatientControllerWebTests` add a few new tests that also verify
the `@Valid` is working:

```java
    @Test
    public void test_PatientController_addPatient_Expect_OK() throws Exception {
        ResponseEntity<Patient> entity = restTemplate.postForEntity("/patients",
                new TestPatientBuilder().build(), Patient.class);

        assertThat(entity.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(entity.getBody()).isNotNull()
                .hasNoNullFieldsOrProperties();
    }

    @Test
    public void test_PatientController_addPatient_WithEmpty_Expect_BadRequest() throws Exception {
        ResponseEntity<String> json = restTemplate.postForEntity("/patients", new Patient(), String.class);

        assertThat(json.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        JSONAssert.assertEquals("{exception:\"org.springframework.web.bind.MethodArgumentNotValidException\"}", json.getBody(), false);
    }

    @Test
    public void test_PatientController_addPatient_WithEmptyGivenName_Expect_BadRequest() throws Exception {
        ResponseEntity<String> json = restTemplate.postForEntity("/patients",
                new TestPatientBuilder().withGivenName("").build(), String.class);

        assertThat(json.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        JSONAssert.assertEquals("{exception:\"org.springframework.web.bind.MethodArgumentNotValidException\"}", json.getBody(), false);
    }

    @Test
    public void test_PatientController_addPatient_WithEmptyFamilyName_Expect_BadRequest() throws Exception {
        ResponseEntity<String> json = restTemplate.postForEntity("/patients",
                new TestPatientBuilder().withFamilyName("").build(), String.class);

        assertThat(json.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        JSONAssert.assertEquals("{exception:\"org.springframework.web.bind.MethodArgumentNotValidException\"}", json.getBody(), false);
    }
```

# Step 13 - More Attributions

The patient needs some more attribution.  Let's add a few new fields that will help round out the `Patient`:

```java
    private String additionalName;
    @Transient
    private Integer age;
    @Email
    private String email;
    @Min(0)
    private Short height; // height in cm
    @Min(0)
    private Short weight; // weight in kg
```

The `additionalName` is useful for capturing a person's middle but what is the @Transient annotation on `age`?  Well
this is a derived attribute based on `birthDate`.  Since it is derived, we don't want JPA to persist it so this
annotation is useful for signifying that (its basically an ignore marker).  However, since we don't have any storage
mechanism for this how will it get set?

I've chosen to calculate the Patient's age each time `birthDate` is set.  Like this:

```java
    public void setBirthDate(LocalDate birthDate) {
        this.birthDate = birthDate;
        this.age = birthDate == null ? null : Period.between(birthDate, LocalDate.now()).getYears();
    }
```

This ensures that any time the birthDate is modified the age is recalculated.  However, if you run the program right now
age will always appear null.  That is a bit of a surprise.  It is because Hibernate is using Field level access based on
how we've used the JPA annotations.  There is a simple way to update the birthDate field so Hibernate will instead use
Property-level access:

```java
    @Access(AccessType.PROPERTY)
    private LocalDate birthDate;
```

Now Hibernate will use the getter/setter methods when accessing birthDate and age will get set appropriately.

In addition add the associated getter and setter for the new properties and restart the application.  Note: since age is
not stored we don't want anyone to accidentally change it.  An easy way to ensure this is to not provide a setter method
for it.

Our new properties are being returned as null in the response JSON.  If we don't want Jackson to marshal null values,
there is a quick setting that we can change in the `application.properties`:

```properties
spring.jackson.default-property-inclusion=non_null
```

That looks better.  If we want, we can go ahead and update the patients.csv to support these new fields:

```csv
ID,BIRTH_DATE,GIVEN_NAME,FAMILY_NAME,ADDITIONAL_NAME,GENDER,EMAIL,HEIGHT,WEIGHT
1,1972-05-05,Phillip,Spec,J,1,pjspec@junit.org,84,180
2,1973-06-06,Sally,Certify,T,2,,78,163
3,1962-02-15,Frank,Neubus,,0,,88,178
```

Then update the associate `data.sql`:

```sql
INSERT INTO PATIENT(ID,BIRTH_DATE,GIVEN_NAME,FAMILY_NAME,ADDITIONAL_NAME,GENDER,EMAIL,HEIGHT,WEIGHT)
  (SELECT * FROM CSVREAD('classpath:patients.csv'));
```

We also need to look at our test cases but first, lets update the `TestPatientBuilder` to account for the new
attributes:

```java
        testPatient.setEmail(String.format("%s@%s.com", RandomStringUtils.randomAlphanumeric(20),
                RandomStringUtils.randomAlphanumeric(20)));
        testPatient.setGender(Gender.values()[RandomUtils.nextInt(0, Gender.values().length)]);
        testPatient.setHeight((short) RandomUtils.nextInt(140, 300));
        testPatient.setWeight((short) RandomUtils.nextInt(50, 90));
```

Don't forget to add the associate "with" methods so we can override the random values if needed.

```java
    public TestPatientBuilder withAdditionalName(String additionalName) {
        testPatient.setAdditionalName(additionalName);
        return this;
    }

    public TestPatientBuilder withEmail(String email) {
        testPatient.setEmail(email);
        return this;
    }

    public TestPatientBuilder withGender(Gender gender) {
        testPatient.setGender(gender);
        return this;
    }

    public TestPatientBuilder withHeight(Short height) {
        testPatient.setHeight(height);
        return this;
    }

    public TestPatientBuilder withWeight(Short weight) {
        testPatient.setWeight(weight);
        return this;
    }
```

With the builder in place add some additional tests to the PatientRepositoryTests to exercise some of the new
constraints:

```java
    @Test
    public void test_PatientRepository_SaveWithInvalidEmail_ExpectException() throws Exception {
        thrown.expect(ConstraintViolationException.class);
        thrown.expectMessage(allOf(containsString("email"), containsString("'not a well-formed email address'")));
        patientRepository.save(new TestPatientBuilder().withEmail("baz").build());
    }

    @Test
    public void test_PatientRepository_SaveWithLessThanMinHeight_ExpectException() throws Exception {
        thrown.expect(ConstraintViolationException.class);
        thrown.expectMessage(allOf(containsString("height"), containsString("'must be greater than or equal to 0'")));
        patientRepository.save(new TestPatientBuilder().withHeight((short) -1).build());
    }

    @Test
    public void test_PatientRepository_SaveWithLessThanMinWeight_ExpectException() throws Exception {
        thrown.expect(ConstraintViolationException.class);
        thrown.expectMessage(allOf(containsString("weight"), containsString("'must be greater than or equal to 0'")));
        patientRepository.save(new TestPatientBuilder().withWeight((short) -1).build());
    }
```

Add a few more tests to the PatientControllerWebTests class:

```java
    @Test
    public void test_PatientController_addPatient_WithInvalidEmail_Expect_BadRequest() throws Exception {
        ResponseEntity<String> json = restTemplate.postForEntity("/patients",
                new TestPatientBuilder().withEmail("bad/email").build(), String.class);

        assertThat(json.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        JSONAssert.assertEquals("{exception:\"org.springframework.web.bind.MethodArgumentNotValidException\"}", json.getBody(), false);
    }

        @Test
    public void test_PatientController_addPatient_WithBirthDate_Expect_ValidAge() throws Exception {
        ResponseEntity<Patient> entity = restTemplate.postForEntity("/patients",
                new TestPatientBuilder().withBirthDate(LocalDate.of(1980, 1, 1)).build(), Patient.class);

        assertThat(entity.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(entity.getBody()).isNotNull()
                .hasFieldOrPropertyWithValue("age", LocalDate.now().getYear() - 1980);
    }
```

# Step X:  Add Remaining "CRUD" Operations to the Patient

So far we have the CR of CRUD implemented.  This seems like a good opportunity to add the remaining lifecycle
operations.  In the `PatientController` add the following code:

```java
    @PutMapping(Patient.RESOURCE_PATH + "/{id}")
    public Patient updatePatientIncludingNulls(@PathVariable("id") Long id, @Valid @RequestBody Patient patient) {
        Patient aPatient = patientRepository.findById(id)
                .orElseThrow(() ->
                        new NotFoundException(String.format("Resource %s/%d not found", Patient.RESOURCE_PATH, id)));
        // copy bean properties including nulls
        BeanUtils.copyProperties(patient, aPatient);
        return patientRepository.save(aPatient);
    }
```

Simply replace the existing resource with the one supplied in the RequestBody.  This implementation first attempts to
load the resource in question and then overlay it with the one provided (including null or missing fields).  The
resource should return the replaced entity at the same location (id).

Add the following to the `PatientControllerWebTests`:

```java
    @Test
    public void test_PatientController_updatePatient_WithValidRandom_Expect_OK() throws Exception {
        HttpEntity<Patient> patientToUpdate = new HttpEntity<>(new TestPatientBuilder().build());
        
        ResponseEntity<Patient> entity = restTemplate.exchange("/patients/{id}", HttpMethod.PUT,
                patientToUpdate, Patient.class, new HashMap<String, Object>() {{ put("id", 1L); }});

        assertThat(entity.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(entity.getBody()).isNotNull()
                .isEqualToIgnoringGivenFields(patientToUpdate.getBody(), "id");
    }
```

# Step X: Returning a Collection of Resources

So far all of our resource interactions are with single entities but we frequently want to retrieve collections of
resources.  Spring Data provides some basic ones for JPA by default, so lets see what that might look like in our
controller.

Add the following code to the PatientController:

```java
    @GetMapping(Patient.RESOURCES_PATH)
    public List<Patient> getPatients() {
        return patientRepository.findAll();
    }
```

If you hit the /patients resource you should see the default patients that we have automatically created with the
data.sql and CSV file.  The result is a raw collection of resources.  The result set is small so we can return the whole
collection with this call but in reality, most unbounded queries like this can cause performance issues if not
automatically bounded.

To add this capability we need to add support for paging.  Replace the previous method with this:

```java
    @GetMapping(Patient.RESOURCE_PATH)
    public List<Patient> getPatients(Pageable pageable) {
        Page<Patient> pagedResults = patientRepository.findAll(pageable);

        if (!pagedResults.hasContent()) {
            throw new NotFoundException(String.format("Resource %s not found", Patient.RESOURCE_PATH));
        }

        return pagedResults.getContent();
    }
```

Now request the collection with paging parameters via http:

```bash
http "localhost:8080/patients?size=1&page=1"
```

The result is now limited to the second result (just remember that page starts at zero).  Without a lot of effort we can
now support paging.  The default page size is 20 so if a user does not provide any paging parameters, we can be assured
that we won't accidentally allow unbounded queries to be returned.  You can customize this with the `@PageableDefault`
annotation but to do so globally isn't possible yet without overriding `WebMvcConfigurerAdapter` and adding a customized
`PageableHandlerMethodArgumentResolver`.

In addition to pagination, you also get sorting support:

```bash
http "localhost:8080/patients?sort=familyName"
```

The result should be sorted as it was requested.  You can also verify this by reviewing the SQL that is being executed
in the logs:

```sql
    select
        patient0_.id as id1_1_,
        patient0_.additional_name as addition2_1_,
        patient0_.birth_date as birth_da3_1_,
        patient0_.email as email4_1_,
        patient0_.family_name as family_n5_1_,
        patient0_.gender as gender6_1_,
        patient0_.given_name as given_na7_1_,
        patient0_.height as height8_1_,
        patient0_.weight as weight9_1_ 
    from
        patient patient0_ 
    order by
        patient0_.family_name asc limit ?
```

TODO: discuss using HTTP header for metadata versus a JSON envelope for paging info.  Research shows lots of examples
using a JSON wrapper but many write-up/guides say very explicitly not to do this arguing that HTTP itself is the
envelope and to use headers for this information.  Right now I am leaning towards headers knowing that this does impact
human readability slightly if you are just using a default browser for GET requests.

To add the pagination metadata to the result replace/add the following code:

```java
    @GetMapping(Patient.RESOURCE_PATH)
    public List<Patient> getPatients(@PageableDefault(size = 30) Pageable pageable, HttpServletResponse response) {
        Page<Patient> pagedResults = patientRepository.findAll(pageable);

        setMetadataResponseHeaders(response, pageable, pagedResults);

        if (!pagedResults.hasContent()) {
            throw new NotFoundException(String.format("Resource %s not found", Patient.RESOURCE_PATH));
        }

        return pagedResults.getContent();
    }

    private void setMetadataResponseHeaders(HttpServletResponse response, Pageable pageable, Page pagedResults) {
        // TODO look into additional meta fields like first and last
        response.setHeader("X-Meta-Pagination",
                String.format("page-number=%d,page-size=%d,total-elements=%d,total-pages=%d",
                        pageable.getPageNumber(), pageable.getPageSize(),
                        pagedResults.getTotalElements(), pagedResults.getTotalPages()));
    }
```

If we make a call to /patients now, we get back a pagination result metadata header:

```
X-Meta-Pagination: page-number=0,page-size=30,total-elements=3,total-pages=1
```

This is pretty useful and non-intrusive on the client.  There are probably a lot of other pieces of metadata that could
prove useful and we will explore them further over time.

# Step X: Change Response to not return "raw" ids

It can be argued that returning the raw database id isn't a good RESTful design (or secure).  We should probably
introduce a UUID instead and use that as the reference.  But first, lets replace the raw id with the path representation
of itself.  To do so is fairly simple; let's modify the Patient entity as such:

```java
    @JsonIgnore
    public Long getId() {
        return id;
    }

    public String getResourceId() {
        return RESOURCE_PATH + "/" + id;
    }
```

The `@JsonIgnore` will cause Jackson to ignore this attribute during marshalling and instead use our synthetic resource
path to the entity.  Our results will now look something like this:

```json
{
  "givenName": "Sally",
  "additionalName": "T",
  "familyName": "Certify",
  "birthDate": "1973-06-06",
  "age": 43,
  "gender": "female",
  "height": 78,
  "weight": 163,
  "resourceId": "/patients/2"
}
```

We could also choose to just ignore the id/resourceId entirely and use a Link header with "self" described but this
won't work quite as well for collection resources.

Sidenote: I also update the JMeter script to run a Find All and rerun the scripts together.  After a few runs to warm
up the JVM we get pretty consistent sub millisecond responses.  So far it does not appear that we have had a negative
impact on performance.

As with most new features lets make sure we have unit tests to support them.  In the PatientControllerWebTest add the
following code:

```java
    @Test
    public void test_PatientController_getPatients_Expect_OK() throws Exception {
        ResponseEntity<Patient[]> entity = restTemplate.getForEntity("/patients", Patient[].class);

        assertThat(entity.getStatusCode()).isEqualTo(HttpStatus.OK);

        assertThat(Arrays.asList(entity.getBody())).isNotNull()
                .extracting(Patient::getFamilyName)
                .contains("Spec", "Certify", "Neubus");
    }

    @Test
    public void test_PatientController_getPatientsWithPagination_Expect_PagedResult() throws Exception {
        ResponseEntity<Patient[]> entity =
                restTemplate.getForEntity("/patients?page={page}&size={size}", Patient[].class, 1, 1);

        assertThat(entity.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(entity.getHeaders().containsKey("X-Meta-Pagination")).isTrue();

        assertThat(Arrays.asList(entity.getBody())).isNotNull()
                .extracting(Patient::getFamilyName)
                .containsOnly("Certify");
    }

    @Test
    public void test_PatientController_getPatientsWithSortDesc_Expect_OrderedResult() throws Exception {
        ResponseEntity<Patient[]> entity =
                restTemplate.getForEntity("/patients?sort=familyName,desc", Patient[].class);

        assertThat(entity.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(entity.getHeaders().containsKey("X-Meta-Pagination")).isTrue();

        assertThat(Arrays.asList(entity.getBody())).isNotNull()
                .extracting(Patient::getFamilyName)
                .contains("Spec", "Neubus", "Certify");
    }
```

# Step X: Refactor Patient to use UUIDs as Primary Keys

After reading several articles and counter articles I decided to switch the Patient table to use a UUID instead of a
sequential Long.

The primary reasoning for this changes was to make it more difficult for someone to guess valid database IDs for
Patients if the system was somehow compromised.  Yes, we will add security around the REST calls and validate that the
user has the authority to see the Patient record, but in case there is a flaw or weakness accidentally introduced into
the system, this will make it quite a bit harder to just scrape the patient records just by guessing the next sequence
number.

The downside of this change is that storing each patient is a little more inefficient and takes more space.  Also,
depending on the actual database used UUIDs may not be stored as efficiently as possible and may have performance
problems at larger scales (> a couple million).  Many of these claims are several years old and database technologies
have improved in this area so it may be worth re-evaluating on a per database perspective.

On a side note, one could argue that a Patient should have some sort of generated universal Patient ID that is unique
and "human manageable" (meaning something one could relate on the phone) but we would need to derive some kind of
generation scheme that makes it reasonably difficult to guess what that ID would be.  Whatever this would look like
would likely have to be driven by government and standards bodies and is outside the scope of this exercise.
  
Even something like SSN as a natural key is somewhat suspect to me as it might give easy cross-system correlation of
data assuming the attacker is just using a secondary table lookup to harvest valid entries from this database.

In all this wasn't terribly difficult.  For the Patient entity we just convert the `@Id` field and the associated
getter methods:

```java
    @Id
    @GeneratedValue
    @Column(columnDefinition = "uuid", updatable = false)
    private UUID id;

    ...

    @JsonIgnore
    public UUID getId() {
        return id;
    }

    public String get_id() {
        return RESOURCE_PATH + "/" + id;
    }
```

Several test cases need to be reworked and the patients.csv needs to be updated to reflect the UUID instead of a number:

```csv
ID,BIRTH_DATE,GIVEN_NAME,FAMILY_NAME,ADDITIONAL_NAME,GENDER,EMAIL,HEIGHT,WEIGHT
e7a47ecd-4182-4209-911b-f7574ded1611,1972-05-05,Phillip,Spec,J,1,pjspec@junit.org,84,180
db890577-b31c-46d6-ae79-da965bcc8d5b,1973-06-06,Sally,Certify,T,2,,78,163
9b107a19-92c8-4242-8cdb-d8c07fd96b15,1962-02-15,Frank,Neubus,,0,,88,178
```

One other last thought is to combine the two ideas and still use a Long as Id and have a secondary indexed field for the
UUID field and only allow GETs on the UUID.  This could lessen the impact on databases that use clustered indexes (and
thus have issues with indexing randomized keys) by making it possible to periodically regenerate the index.  I still
suspect there may be performance issues with millions of rows however...

TODO: Need to rewrite this section and maybe earlier since I decided on combining the ideas (with the
UUID being just a unique secondary field).

# Step X: Refactor Paging Header Metadata with a Custom ResponseBodyAdvisor

Generally, we want to avoid a lot of boilerplate code and keep with the DRY principle.  I was discovering
that the paging metadata headers were going to be causing a lot of boilerplate.  I examined using inheritance
on Controllers and Util classes but decided on a feature of Spring MVC that lets you create custom
"advisors" that can intercept reponses of specific types and shape them as needed prior to Jackson
marshalling.

Create a class called `PageResponseBodyAdvisor` with the code:

```java
@ControllerAdvice
public class PageResponseBodyAdvisor extends AbstractMappingJacksonResponseBodyAdvice {

  public static final String PAGE_METADATA_FMT = "page-number=%d,page-size=%d,total-elements=%d,total-pages=%d,first-page=%b,last-page=%b";

  @Override
  public boolean supports(MethodParameter returnType,
      Class<? extends HttpMessageConverter<?>> converterType) {
    return super.supports(returnType, converterType) &&
        Page.class.isAssignableFrom(returnType.getParameterType());
  }

  @Override
  protected void beforeBodyWriteInternal(MappingJacksonValue bodyContainer, MediaType contentType,
      MethodParameter returnType, ServerHttpRequest request, ServerHttpResponse response) {

    Page<?> page = ((Page<?>)bodyContainer.getValue());

    response.getHeaders().add("X-Meta-Pagination",
        String.format(PAGE_METADATA_FMT, page.getNumber(), page.getSize(), page.getTotalElements(),
            page.getTotalPages(), page.isFirst(), page.isLast()));

    // finally, strip out the actual content and replace it as the body value
    bodyContainer.setValue(page.getContent());
  }
}
```

This handily replaces the responses of type Page<?> by replacing the MappingJacksonValue container
with the actual response type (which is a Collection of things) and uses the Page wrapper to create
the custom Header metadata fields.

Now by refactoring the existing code to return the raw Page result from the find queries we no longer
have to worry about manually handling the specific pageable result.  We'll probably be able to use
this mechanism again later when we want to add more header metadata or shape the responses in some
way.

# Step X: Add finder with Query By Example

Spring Data recently added support for Query By Example.  Basically, QBE is a type of query that is
dynamically built based on a reference object.  Fields that are provided are used in the resulting
WHERE clause.  Typically, this can be rather limiting as many matches you want to use LIKE or greater
than/less than which is why the API also allows for a customizable "Example Matcher" where you can
add greater customization for specific fields.

By default, you don't have to do anything extra to have the QBE operations on your Repositories,
however, you'll probably want to implent them something like this:

```java
  @GetMapping(Patient.RESOURCE_PATH + "/queryByExample")
  public Page<Patient> getPatientsByExample(@RequestParam Map<String, Object> paramMap,
      @PageableDefault(size = 30) Pageable pageable, ObjectMapper objectMapper) {
    // TODO doesn't seem to handle the LocalDate conversion
    // copy the map of query params into a new instance of the Patient POJO
    Patient examplePatient = objectMapper.convertValue(paramMap, Patient.class);

    Page<Patient> pagedResults = patientRepository
        .findAll(Example.of(examplePatient, DEFAULT_MATCHER.withIgnorePaths("patientId")), pageable);

    if (!pagedResults.hasContent()) {
      throw new NotFoundException(String.format("Resource %s not found", Patient.RESOURCE_PATH));
    }

    return pagedResults;
  }
```

This is taking all passed in request parameters and putting them in a Map and then using a Jackson
convenience method copying them into a POJO of the type specified.  This creates the "example" object
that is used in the query construction.  Only fields that are provided are mapped and the Example.of
only uses non-null values for matching.

In this case I'm using a default matcher that is defined like this:

```java
  private static final ExampleMatcher DEFAULT_MATCHER = ExampleMatcher.matching()
      .withStringMatcher(CONTAINING)
      .withIgnoreCase();
```

This is telling Spring Data to construct an example query that matches on any String property as a
"contains" (like %string%) and to ignore case.  This is a frequent user requirement for searches.
However, I discovered a slight side-effect from the UUID field that posed problems.  Since the UUID
is set as part of the constructor, it has a non-null value and, because it is unique it would cause
all queries to fail to return results.  Adding the "withIgnorePath" causes the property to be ignored
even if it has a value.  This is a bit of a compromise but I think the extra verboseness isn't too
annoying.

Also, unfortunately my trick with the Jackson object mapper fails to work for LocalDates (and
probably most of the Joda/Java8 dates).  I think this must be a bug in Jackson but it makes it
impossible to map dates to example queries at this time without work-around code.

TODO need to isolate the issue with Jackson and submit a bug request.

# Step X: Write tests for Query By Example


---

## TODOs

- Add custom @Valid
- Add more example mocking tests (when and why)
- Add equals/hashcodes
- Add better performance tests
- Add Query by Example examples
- Add QueryDsl support
- Add Optimistic Locking support with @Version
- Add more data to the patients.csv
- Add custom response wrapping
- Add support for Flyway
- Add support for JPA/JTA transactions
- Add REST documentation Swagger vs RESTDocs (http://apihandyman.io/categories/posts/)
- Add Spring Security with OAuth2 and JWT
- Add Auditing
- Explain HIPA and PII concerns
- Convert to Kotlin?
- Discuss this JPA approach benefits has initial schema-less feel
- Research https://github.com/FasterXML/jackson-datatype-hibernate
- Google style https://github.com/google/styleguide


# Additional Resources

- [Spring Boot Reference Guide](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Data JPA](http://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Hibernate ORM](http://hibernate.org/orm/)
- [Hibernate Validator](http://hibernate.org/validator/)
- [Jackson](http://wiki.fasterxml.com/JacksonHome)
- [H2 Database](http://www.h2database.com/html/main.html)
- [AssertjJ](https://joel-costigliola.github.io/assertj/)

# Reference REST design guides:
- https://github.com/Microsoft/api-guidelines/blob/master/Guidelines.md
- https://docs.microsoft.com/en-us/azure/best-practices-api-design


# Good Blogs I've found on this journey

- https://vladmihalcea.com/
- https://www.petrikainulainen.net/blog/

# License

    Copyright 2016-2017 Shawn Sherwood

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

        http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
