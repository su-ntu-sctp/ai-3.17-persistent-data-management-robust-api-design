# Lesson Title
Persistent Data Management and Robust API Design

## Lesson Overview
This lesson extends the `simple-crm` project by switching from H2 to PostgreSQL, introducing JPA-derived queries, and strengthening the API through global exception handling and input validation. Learners will build reliable, production-ready APIs with persistent data storage and consistent error responses.

## Lesson Objectives
- Configure PostgreSQL as the application database using JPA.  
- Implement derived queries for flexible data retrieval.  
- Use `Optional` and custom exceptions for safer lookups.  
- Centralize error handling with a Global Exception Handler.  
- Apply validation annotations to enforce input integrity.  
This lesson is the 2nd part of JPA, where we switch from H2 to PostgreSQL and make our application more robust with exception handling and validation.

---

## Part 1: PostgreSQL

PostgreSQL is a relational database management system (RDBMS). It is open source and free. It is also known as Postgres and it is popular for its reliability, robustness, and performance.

Check if you have PostgreSQL installed:

```sh
psql --version
```

### Installation on Windows (WSL)

First, update your system:

```sh
sudo apt update && sudo apt upgrade
```

Install PostgreSQL:

```sh
sudo apt install postgresql postgresql-contrib
```

Managing the PostgreSQL service:

```sh
sudo service postgresql start
sudo service postgresql stop
sudo service postgresql restart
sudo service postgresql status
```

Enter PostgreSQL shell:

```sh
psql -U postgres
```

This will log you in as the `postgres` user. By default, a **superuser** named `postgres` is created when PostgreSQL is installed.

### Installation on Mac

To install PostgreSQL on Mac, we will be using Homebrew. Homebrew is a package manager for Mac. It is similar to `apt-get` on Ubuntu.

Install Homebrew:

```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

If you have installed Homebrew before, you can update it with:

```sh
brew update && brew upgrade
```

Install PostgreSQL using Homebrew:

```sh
brew install postgresql@16
```

Managing the PostgreSQL service:

```sh
brew services start postgresql@16
brew services stop postgresql@16
brew services restart postgresql@16
brew services info postgresql@16
```

Enter PostgreSQL shell:

```sh
psql postgres
```

This starts the PostgreSQL shell and connects to the `postgres` database. By default, no password is needed to connect to the database. For the Homebrew installation, we do not need to specify the username because it uses the current user by default.

Unlike the WSL installation, the Homebrew installation does not create a superuser named `postgres`. Instead, it uses your Mac username as the default username. You can check your Mac username with:

```sh
whoami
```

### Dbeaver

DBeaver is a free and open-source universal database tool for developers and database administrators. It is a GUI program that allows you to connect to a database and perform database operations easily.

Download and install DBeaver from [here](https://dbeaver.io/download/).

Create a new connection to PostgreSQL:

<img src="./assets/images/Postgres New Connection.png" width=500>

Configure the connection settings:

<img src="./assets/images/Connection Settings.png" width=500>

For WSL, the username (`postgres`) and password should be the same as what you have configured during the installation of PostgreSQL. Or it can be left blank if you have set the authentication method to `trust` in the `pg_hba.conf` file (instructions below).

For Mac, the default username is your Mac username and no password is needed.

This can be changed in the `pg_hba.conf` file as needed.

### Configuring PostgreSQL Client Authentication (Optional)

To connect to PostgreSQL, you may need to configure the client authentication method. This can be done by editing the `pg_hba.conf` file.

The location of the `pg_hba.conf` file depends on your system.

- WSL: `/etc/postgresql/16/main/pg_hba.conf`
- Homebrew (Intel): `/usr/local/var/postgres/`
- Homebrew (Apple Silicon): `/opt/homebrew/var/postgresql@16/pg_hba.conf`

You can use the `vi` or `nano` command to edit the `pg_hba.conf` file.

```sh
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

The `pg_hba.conf` file contains a list of records. Each record specifies a connection type, a database name, a user name, an IP address range, and an authentication method.

Depending on your system, the default authentication method might be `scram-sha-256`. This means that you need to provide a password to connect to the database. If you set the authentication method to `trust`, you don't need to provide a password to connect to the database. For more info on the `pg_hba.conf` file, you can read [here](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html).

```sh
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
```

If you have forgotten your password, you can set the authentication method to `trust`. Then you can log in to the database without a password to change it. Remember to update `pg_hba.conf` and set the authentication method back to `scram-sha-256`. Note that PostgreSQL service needs to be restarted after editing the `pg_hba.conf` file.

```
psql -U postgres
postgres=# \password
```

You can read more about configuring the `pg_hba.conf` file [here](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html).

### Adding a New Database

With PostgreSQL installed, we can now create a new database for our application.

This can be done using DBeaver or using psql shell.

```sql
CREATE DATABASE simple_crm;
```

In DBeaver, right click on "Databases" and select "Create Database". Enter the database name and click "OK".

---

## Part 2: Switching to PostgreSQL

Open the `simple-crm` project.
With JPA, we can easily switch to another database.

First, we need to add the PostgreSQL dependency to our project.

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

You can comment out the H2 dependency for now.

```xml
<!-- <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
</dependency> -->
```

Next, we need to update the `application.properties` file.

```properties
# Comment out the H2 configuration
# H2
# spring.h2.console.enabled=true
# spring.h2.console.path=/h2
# spring.datasource.url=jdbc:h2:mem:simple-crm-h2

# PostgreSQL
spring.datasource.url=jdbc:postgresql://localhost:5432/simple_crm
# for WSL, use postgres
# for Mac, use your Mac username
spring.datasource.username=postgres
# Password can be blank if we set it to trust in pg_hba.conf
spring.datasource.password=
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
# this will drop and create tables again
spring.jpa.hibernate.ddl-auto=create
# this can be used to update tables
# spring.jpa.hibernate.ddl-auto=update
```

Start your application and test the endpoints.

Check the tables in DBeaver.

You can see that with JPA, we can easily switch to another database without changing our code.

---

## Part 3: JPA Query Creation from Method Name

JPA provides a set of [methods](https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/JpaRepository.html) for us to perform CRUD operations. However, there may be times when we need to perform more complex queries. For example, we may want to find all customers with a certain first name.

In this case, we can create a query using the method name. This is known as JPA Query Creation from Method Name.

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {

    // Custom query to find all customers with a certain first name
    List<Customer> findByFirstName(String firstName);
}
```

In our service layer `CustomerServiceImpl`, we can call this method. Remember to add the method to the interface `CustomerService` too.

```java
@Override
public ArrayList<Customer> searchCustomers(String firstName) {
    List<Customer> foundCustomers = customerRepository.findByFirstName(firstName);
    return (ArrayList<Customer>) foundCustomers;
}
```

And in our controller, we can create a new endpoint to allow the user to search for customers by first name.

```java
@GetMapping("/search")
public ResponseEntity<ArrayList<Customer>> searchCustomers(@RequestParam String firstName) {
    ArrayList<Customer> foundCustomers = customerService.searchCustomers(firstName);
    return new ResponseEntity<>(foundCustomers, HttpStatus.OK);
}
```

Try to search for customers by first name.

`http://localhost:8080/customers/search?firstName=Stephen`

To search for first names starting with a certain string, we can use the `StartingWith` keyword. You can try this on your own in your free time.

```java
List<Customer> findByFirstNameStartingWith(String firstName);
```

For more information, you can read more about [JPA Derived Query from Method Name](https://www.baeldung.com/spring-data-derived-queries).

---

## Part 4: `Optional` and Exception Handling

Currently, in our `CustomerServiceImpl`, we are doing this:

```java
@Override
public Customer getCustomer(Long id) {
    Customer foundCustomer = customerRepository.findById(id).get();
    return foundCustomer;
}
```

What happens when we get a customer id that does not exist?

If you look at the `findById` method in the `CustomerRepository` interface, you will see that it actually returns an `Optional`.

An `Optional` is a container object that may or may not contain a non-null value. It is used to represent the presence or absence of a value.

Hence, we had to use `get()` to unwrap the `Optional` and get the `Customer` object. However, this is not a good practice. If the result is `null`, we will get a `NoSuchElementException`. You can try getting an invalid id.

We should check if the `Optional` contains a value before unwrapping it.

```java
@Override
public Customer getCustomer(Long id) {
    Optional<Customer> optionalCustomer = customerRepository.findById(id);
    if (optionalCustomer.isPresent()) {
        // If the Optional contains a value, unwrap it and return the Customer object
        Customer foundCustomer = optionalCustomer.get();
        return foundCustomer;
    }

    throw new CustomerNotFoundException(id);
}
```

Test getting an invalid id.

This can also be further simplified with `orElseThrow()`.

The method `customerRepository.findById(id)` will return an `Optional<Customer>`. `Optional` has a method `orElseThrow()` that will return the value if it is present, or throw an exception if it is not present. We can then use a lambda expression to throw a `CustomerNotFoundException` if the value is not present.

```java
@Override
public Customer getCustomer(Long id) {
    return customerRepository.findById(id).orElseThrow(() -> new CustomerNotFoundException(id));
}
```

You can use whichever method you think is more readable.

Currently, we are only getting a 404 error. We are not getting a proper error message, which may not be very helpful to the user.

We could return the error message as a string, but our `ResponseEntity` is of type `Customer`. So we could change the type parameter of `ResponseEntity` to `Object`, which is the parent class of all classes in Java.

```java
@GetMapping("{id}")
public ResponseEntity<Object> getCustomer(@PathVariable Long id) {

    try {
        Customer foundCustomer = customerService.getCustomer(id);
        return new ResponseEntity<>(foundCustomer, HttpStatus.OK);
    } catch (CustomerNotFoundException e) {
        return new ResponseEntity<>(e.getMessage(), HttpStatus.NOT_FOUND);
    }
}
```

With this, we could return a `Customer` object if the customer is found, or a string if the customer is not found. But we lose the type safety. This means we could return any object, not just a `Customer` object.

Another problem we have now is that we have all these `try-catch` blocks in our controller.

---

## Part 5: Global Exception Handler

Spring Boot lets us create a global exception handler to handle all exceptions in our application using the `@ControllerAdvice` annotation. This allows us to define a centralized place to handle exceptions thrown from all controllers in our application.

<img src="https://miro.medium.com/v2/resize:fit:1100/format:webp/1*Rr3r5KfKYc6fVJfTZF-rHA.png" width=550 style="background-color: #fff;padding: 25px; border: 1px solid #333;border-radius: 5px">

> Source: https://medium.com/@praneshgupta/springboot-exception-handling-in-apis-globalexceptionhandler-c549470f7834

Create a new class `GlobalExceptionHandler.java`.

Each exception handler method can be annotated with `@ExceptionHandler` to handle a specific exception.

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

@ControllerAdvice
public class GlobalExceptionHandler {

    // This is a handler for CustomerNotFoundException
    @ExceptionHandler(CustomerNotFoundException.class)
    public ResponseEntity<String> handleCustomerNotFoundException(CustomerNotFoundException ex) {
        return new ResponseEntity<>(ex.getMessage(), HttpStatus.NOT_FOUND);
    }
}
```

Hence we no longer need the `try-catch` block in our controller. And we also do not have to return a `ResponseEntity<Object>` and instead return our original `ResponseEntity<Customer>`, which gives us type safety.

```java
@GetMapping("{id}")
public ResponseEntity<Customer> getCustomer(@PathVariable Long id) {
    Customer foundCustomer = customerService.getCustomer(id);
    return new ResponseEntity<>(foundCustomer, HttpStatus.OK);
}
```

Now, when an exception is thrown, it will be caught by this global exception handler. The specific exception handler method will be called to handle the exception.

Currently we are returning it as a string. We could give it a more proper structure by creating a new `ErrorResponse` class .

```java
@Getter
@Setter
@AllArgsConstructor
public class ErrorResponse {
  private String message;
  private LocalDateTime timestamp;
}
```

And update the exception handler method to return an `ErrorResponse` object.

```java
@ExceptionHandler(CustomerNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleCustomerNotFoundException(CustomerNotFoundException ex) {
    ErrorResponse errorResponse = new ErrorResponse(ex.getMessage(), LocalDateTime.now());
    return new ResponseEntity<>(errorResponse, HttpStatus.NOT_FOUND);
}
```

Providing a meaningful error message is useful for our frontend too as it can display the error message to the user.

### General Exception Handler

We can also add a general exception handler to handle all other exceptions that are not handled by the specific exception handlers. This is useful because there may be other exceptions that we have not yet handled or that we did not expect.

For example, currently, we have not handled invalid interactions. Try to get an invalid interaction.

We will get a `NoSuchElementException` when we try to get an interaction that does not exist.

With a general exception handler, we can return a generic error message to the user.

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleException(Exception ex) {
    // We can log the exception here
    // logger.error(ex.getMessage(), ex);
    // Return a generic error message
    ErrorResponse errorResponse = new ErrorResponse("Something went wrong", LocalDateTime.now());
    return new ResponseEntity<>(errorResponse, HttpStatus.INTERNAL_SERVER_ERROR);
}
```

Try to get an invalid interaction again.

With this in place, the user will get a generic error message when an exception is thrown and we do not have to expose too much information about our application.

We can add another exception handler to handle this exception.

### 👨‍💻 Activity

Refactor the code in `CustomerServiceImpl` and `InteractionServiceImpl` to use `Optionals`. Create a new `InteractionNotFoundException` class that extends `RuntimeException`. Throw this exception when the interaction is not found.

Use the Global Exception Handler to handle these exceptions.

### Handling more than one exception

You may have noticed that the handler method for both `CustomerNotFoundException` and `InteractionNotFoundException` is almost identical. How can we refactor it then?

In the `@ExceptionHandler` annotation, we can use a array notation to define a array of exception class to handle. For the method, we will need to define a parameter of common type between the exception classes. In this case, `CustomerNotFoundException` and `InteractionNotFoundException` are both `RuntimeException`.

By changing the `handleCustomerNotFoundException` method as shown below, we can further refactor the codes by minimising the number of handlers method we need.

```java
// rename handleCustomerNotFoundException -> handleResourceNotFoundException
@ExceptionHandler({CustomerNotFoundException.class, InteractionNotFoundException.class})
    public ResponseEntity<ErrorResponse> handleResourceNotFoundException(RuntimeException ex) {
    ErrorResponse errorResponse = new ErrorResponse(ex.getMessage(), LocalDateTime.now());
    return new ResponseEntity<>(errorResponse, HttpStatus.NOT_FOUND);
}

// @ExceptionHandler(InteractionNotFoundException.class)
//    public ResponseEntity<ErrorResponse> handleInteractionNotFoundException(InteractionException ex) {
//    ErrorResponse errorResponse = new ErrorResponse(ex.getMessage(), LocalDateTime.now());
//    return new ResponseEntity<>(errorResponse, HttpStatus.NOT_FOUND);
//}

```

---

## Part 6: Validation

Try to create a new customer with an empty name and an invalid email address.

We have a problem now because the user could input invalid data such as an empty name or an invalid email address.

This should be handled by the frontend. But we cannot assume that it will always be handled or that it is handled correctly. Hence, we should also validate the data on the backend to prevent invalid data from being saved to the database.

To do this, let's install the `spring-boot-starter-validation` dependency.

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

Let's add validation constraints to our `Customer` class.

```java
public class Customer {
  // ...

  @NotBlank(message = "First name is mandatory")
  private String firstName;

  @Email(message = "Email should be valid")
  private String email;

}
```

This will validate that the `firstName` is not blank and that the `email` is a valid email address.

We also need to use the `@Valid` annotation in our controller. The `@Valid` annotation tells Spring to validate the request body.

```java
@PostMapping
public ResponseEntity<Customer> createCustomer(@Valid @RequestBody Customer customer) {
    Customer createdCustomer = customerService.createCustomer(customer);
    return new ResponseEntity<>(createdCustomer, HttpStatus.CREATED);
}
```

Note that `@Valid` has to be placed before `@RequestBody`. The order matters because `@Valid` tells Spring to validate the request body. If it is placed after `@RequestBody`, Spring will not know what to validate.

Test out the validation.

The validation exception will get caught by our general exception handler.

### `BindingResult`

We can get the validation errors from the `BindingResult` object. The `BindingResult` object contains the result of the validation and any errors that may have occurred during the validation.

```java
@PostMapping("")
public ResponseEntity<Customer> createCustomer(@Valid @RequestBody Customer customer, BindingResult bindingResult) {

    if (bindingResult.hasErrors()) {
        return new ResponseEntity<>(HttpStatus.BAD_REQUEST);
    }

    Customer newCustomer = customerService.createCustomer(customer);
    return new ResponseEntity<>(newCustomer, HttpStatus.CREATED);
}
```

### Catching Validation Exceptions

Since we have a global exception handler now, we can add another exception handler to handle validation exceptions. Remember to remove the `BindingResult` parameter from the controller method first.

```java
// VALIDATION EXCEPTION HANDLER
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidationExceptions(MethodArgumentNotValidException ex) {

    // Get a list of all validation errors from the exception object
    List<ObjectError> validationErrors = ex.getBindingResult().getAllErrors();

    // Create a StringBuilder to store all error messages
    StringBuilder sb = new StringBuilder();

    // Loop through all errors and append error messages to StringBuilder
    for (ObjectError error : validationErrors) {
        sb.append(error.getDefaultMessage() + ". ");
    }

    ErrorResponse errorResponse = new ErrorResponse(sb.toString(), LocalDateTime.now());
    return new ResponseEntity<>(errorResponse, HttpStatus.BAD_REQUEST);
}
```

Sidenote: `StringBuilder` is a mutable sequence of characters. It is more efficient than `String` when we need to append strings. You can read more about `String` vs `StringBuilder` [here](https://medium.com/@AlexanderObregon/understanding-string-vs-stringbuilder-in-java-50448cbbf253).

Test out the validation.

## 👨‍💻 Activity

Using annotations, add validation constraints for the folowing:

- Customer `yearOfBirth` should be between 1940 and 2005 (you'll have to update the dataloader and constructor for this)
- Customer `contactNo` should be 8 characters long (no need to check if it is a valid phone number)
- Interaction `remarks` should be at least 3 characters long and at most 30 characters long
- `interactionDate` should not be in the future

Validation annotations references:

- https://www.baeldung.com/java-validation
- https://education.launchcode.org/java-web-development/chapters/spring-model-validation/validation-annotations.html
- https://cheatsheetseries.owasp.org/cheatsheets/Bean_Validation_Cheat_Sheet.html

---

## Part 7: Install PostgreSQL Using Docker (Optional)

You can also install PostgreSQL using Docker. You can try this on your own in your free time.

Docker is a containerization platform that allows you to run applications in a container. It is similar to a virtual machine but it is more lightweight and faster. You can read more about Docker [here](https://www.docker.com/resources/what-container). Docker will be covered in more details in the DevOps module.

If you don't have Docker installed, you can install it from [here](https://docs.docker.com/get-docker/).

If you have Docker installed, you can install PostgreSQL using Docker.

```sh
docker run --name mypostgres -e POSTGRES_PASSWORD=password -d -p 5433:5432 postgres
```

The command above will pull the latest PostgreSQL image from Docker Hub and run it in a container. The container will be named `mypostgres` and the password for the default user `postgres` will be `password`. The container will be running on port `5433`.

You can check if the container is running with:

```sh
docker ps
```

You can stop the container with:

```sh
docker stop mypostgres
```

You can start the container with:

```sh
docker start mypostgres
```

You can remove the container with:

```sh
docker rm mypostgres
```

---

END