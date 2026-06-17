# Lesson 3.17: Persistent Data Management and Robust API Design

## Lesson Overview
This lesson extends the `simple-crm` project by switching from H2 to PostgreSQL, introducing JPA-derived queries, and strengthening the API through global exception handling and input validation. Learners will build reliable, production-ready APIs with persistent data storage and consistent error responses.

## Starting Point
Before beginning this lesson, your `simple-crm` project should have:
- `Customer` and `Interaction` entity classes with a `@ManyToOne` / `@OneToMany` relationship
- `CustomerRepository` and `InteractionRepository` interfaces extending `JpaRepository`
- `CustomerServiceImpl` with both repositories injected
- H2 configured in `application.properties`
- Basic CRUD operations working for both `Customer` and `Interaction`

If you are missing any of the above, revisit Lesson 3.16 before continuing.

## Lesson Objectives
By the end of this lesson, students will be able to:

1. **Switch** from H2 to PostgreSQL and configure Spring Data JPA for persistent storage
2. **Implement** JPA derived queries and use `Optional` for safer data retrieval
3. **Centralise** error handling and enforce input validation to build a robust, production-ready API

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

To install PostgreSQL on Mac, we will be using Homebrew. Homebrew is a package manager for Mac.

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

This starts the PostgreSQL shell and connects to the `postgres` database. For the Homebrew installation, no password is needed by default, and it uses your Mac username rather than `postgres`. You can check your Mac username with:

```sh
whoami
```

### DBeaver

DBeaver is a free and open-source universal database tool for developers and database administrators. It allows you to connect to a database and perform operations easily via a GUI.

Download and install DBeaver from [here](https://dbeaver.io/download/).

Create a new connection to PostgreSQL:

<img src="./assets/images/Postgres New Connection.png" width=500>

Configure the connection settings:

<img src="./assets/images/Connection Settings.png" width=500>

For WSL, the username (`postgres`) and password should match what you configured during installation. For Mac, the default username is your Mac username and no password is needed.

### Configuring PostgreSQL Client Authentication (Optional)

To connect to PostgreSQL, you may need to configure the client authentication method by editing the `pg_hba.conf` file.

The location depends on your system:
- WSL: `/etc/postgresql/16/main/pg_hba.conf`
- Homebrew (Intel): `/usr/local/var/postgres/`
- Homebrew (Apple Silicon): `/opt/homebrew/var/postgresql@16/pg_hba.conf`

```sh
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

If you set the authentication method to `trust`, you don't need a password to connect. For more info, read [here](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html).

```sh
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
```

Note that the PostgreSQL service needs to be restarted after editing the file.

### Adding a New Database

Create a new database for our application using DBeaver or the psql shell:

```sql
CREATE DATABASE simple_crm;
```

In DBeaver, right-click on "Databases" and select "Create Database".

---

## Part 2: Switching to PostgreSQL

Open the `simple-crm` project. With JPA, switching databases requires minimal code changes.

First, add the PostgreSQL dependency to `pom.xml`:

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

Comment out the H2 dependency:

```xml
<!-- <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
</dependency> -->
```

Update `application.properties`:

```properties
# Comment out the H2 configuration
# spring.h2.console.enabled=true
# spring.h2.console.path=/h2
# spring.datasource.url=jdbc:h2:mem:simple-crm

# PostgreSQL
spring.datasource.url=jdbc:postgresql://localhost:5432/simple_crm
# For WSL use: postgres
# For Mac use: your Mac username
spring.datasource.username=postgres
# Leave blank if authentication is set to trust in pg_hba.conf
spring.datasource.password=

# ⚠️ WARNING: 'create' drops and recreates ALL tables on every startup — all existing data will be lost.
# This is acceptable during early development, but switch to 'update' as soon as you want to keep your data.
spring.jpa.hibernate.ddl-auto=create
# Use this instead when you want to keep existing data:
# spring.jpa.hibernate.ddl-auto=update
# In production, use a migration tool like Flyway or Liquibase instead of ddl-auto.
```

> **Note:** Spring Boot 3.x with Hibernate 6 auto-detects the PostgreSQL dialect — you do not need to set `spring.jpa.database-platform` manually. Omitting it is the recommended approach.

Start the application and test the endpoints. Check the tables in DBeaver.

Notice how we switched databases without changing any of our Java code — this is the power of JPA's database abstraction.

---

## Part 3: JPA Query Creation from Method Name

JPA provides built-in methods for CRUD operations, but there are times when we need more specific queries. For example, finding all customers with a certain first name.

JPA allows us to create queries simply by naming our repository methods in a specific pattern. This is known as **derived queries**.

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {

    // Custom query to find all customers with a certain first name
    List<Customer> findByFirstName(String firstName);
}
```

Spring JPA reads the method name and automatically generates the appropriate SQL query — no SQL needed.

Add a corresponding method to `CustomerServiceImpl`. Remember to add the signature to the `CustomerService` interface too.

```java
@Override
public List<Customer> searchCustomers(String firstName) {
    return customerRepository.findByFirstName(firstName);
}
```

Add a new endpoint in `CustomerController`:

```java
@GetMapping("/search")
public ResponseEntity<List<Customer>> searchCustomers(@RequestParam String firstName) {
    List<Customer> foundCustomers = customerService.searchCustomers(firstName);
    return new ResponseEntity<>(foundCustomers, HttpStatus.OK);
}
```

Test the endpoint:

```
http://localhost:8080/customers/search?firstName=Stephen
```

You can also search for first names starting with a certain string using the `StartingWith` keyword — try this on your own:

```java
List<Customer> findByFirstNameStartingWith(String firstName);
```

For more information, read about [JPA Derived Query from Method Name](https://www.baeldung.com/spring-data-derived-queries).

---

## Part 4: `Optional` and Exception Handling

Currently, in `CustomerServiceImpl` we are doing this:

```java
@Override
public Customer getCustomer(Long id) {
    Customer foundCustomer = customerRepository.findById(id).get();
    return foundCustomer;
}
```

What happens when we request a customer id that does not exist?

If you look at the `findById` method, it actually returns an `Optional<Customer>`. An `Optional` is a container object that may or may not contain a non-null value — it represents the presence or absence of a value.

We used `.get()` to unwrap the `Optional`, but if the value is absent, this throws a `NoSuchElementException`. Try requesting an invalid id and observe the error.

We should check whether the `Optional` contains a value before unwrapping it:

```java
@Override
public Customer getCustomer(Long id) {
    Optional<Customer> optionalCustomer = customerRepository.findById(id);
    if (optionalCustomer.isPresent()) {
        return optionalCustomer.get();
    }
    throw new CustomerNotFoundException(id);
}
```

This can be further simplified using `orElseThrow()`:

```java
@Override
public Customer getCustomer(Long id) {
    return customerRepository.findById(id).orElseThrow(() -> new CustomerNotFoundException(id));
}
```

Use whichever you find more readable. The result is the same — a meaningful `CustomerNotFoundException` instead of a cryptic `NoSuchElementException`.

Currently we only return a `404` status with no message body. To return a proper error message, we could change the return type to `ResponseEntity<Object>`:

```java
@GetMapping("/{id}")
public ResponseEntity<Object> getCustomer(@PathVariable Long id) {
    try {
        Customer foundCustomer = customerService.getCustomer(id);
        return new ResponseEntity<>(foundCustomer, HttpStatus.OK);
    } catch (CustomerNotFoundException e) {
        return new ResponseEntity<>(e.getMessage(), HttpStatus.NOT_FOUND);
    }
}
```

This works, but it loses type safety and clutters our controller with `try-catch` blocks. There is a better way.

---

## Part 5: Global Exception Handler

Spring Boot lets us create a global exception handler using the `@RestControllerAdvice` annotation. This gives us a single centralised place to handle exceptions thrown from any controller in the application — no more scattered `try-catch` blocks.

> **Note:** We use `@RestControllerAdvice` (not `@ControllerAdvice`) for REST APIs. It combines `@ControllerAdvice` and `@ResponseBody`, ensuring all handler methods automatically serialize their return value to JSON — which is exactly what we want in a REST API.

<img src="https://miro.medium.com/v2/resize:fit:1100/format:webp/1*Rr3r5KfKYc6fVJfTZF-rHA.png" width=550 style="background-color: #fff;padding: 25px; border: 1px solid #333;border-radius: 5px">

> Source: https://medium.com/@praneshgupta/springboot-exception-handling-in-apis-globalexceptionhandler-c549470f7834

Create a new class `GlobalExceptionHandler.java`:

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CustomerNotFoundException.class)
    public ResponseEntity<String> handleCustomerNotFoundException(CustomerNotFoundException ex) {
        return new ResponseEntity<>(ex.getMessage(), HttpStatus.NOT_FOUND);
    }
}
```

With this in place, we can remove the `try-catch` block from our controller and restore the original return type:

```java
@GetMapping("/{id}")
public ResponseEntity<Customer> getCustomer(@PathVariable Long id) {
    Customer foundCustomer = customerService.getCustomer(id);
    return new ResponseEntity<>(foundCustomer, HttpStatus.OK);
}
```

When `CustomerNotFoundException` is thrown anywhere in the application, it will be caught and handled by the global handler automatically.

### Structured Error Response

Currently we are returning the error as a plain string. We can give it a proper structure by creating an `ErrorResponse` class.

```java
import java.time.LocalDateTime;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
@AllArgsConstructor
public class ErrorResponse {
    private String message;
    private LocalDateTime timestamp;
}
```

> **Note:** `LocalDateTime` comes from `java.time.LocalDateTime` — make sure this import is present. Your IDE should add it automatically, but check if you see a red squiggle.

Update the exception handler to return an `ErrorResponse`:

```java
@ExceptionHandler(CustomerNotFoundException.class)
public ResponseEntity<ErrorResponse> handleCustomerNotFoundException(CustomerNotFoundException ex) {
    ErrorResponse errorResponse = new ErrorResponse(ex.getMessage(), LocalDateTime.now());
    return new ResponseEntity<>(errorResponse, HttpStatus.NOT_FOUND);
}
```

This gives the frontend a consistent, structured error response it can parse and display meaningfully.

### General Exception Handler

We should also add a catch-all handler for any unexpected exceptions we have not explicitly handled. For example, try requesting an interaction that does not exist — you will get a `NoSuchElementException`.

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleException(Exception ex) {
    // Log the exception here if needed
    ErrorResponse errorResponse = new ErrorResponse("Something went wrong", LocalDateTime.now());
    return new ResponseEntity<>(errorResponse, HttpStatus.INTERNAL_SERVER_ERROR);
}
```

This prevents internal implementation details from leaking to the client while still returning a meaningful response.

### 👨‍💻 Activity **(15 minutes)**

Refactor `CustomerServiceImpl` and `InteractionServiceImpl` to use `orElseThrow()`. Create a new `InteractionNotFoundException` class that extends `RuntimeException`. Throw it when an interaction is not found.

Update the `GlobalExceptionHandler` to handle both `CustomerNotFoundException` and `InteractionNotFoundException`. Since both extend `RuntimeException`, you can handle them in a single method using array notation:

```java
@ExceptionHandler({CustomerNotFoundException.class, InteractionNotFoundException.class})
public ResponseEntity<ErrorResponse> handleResourceNotFoundException(RuntimeException ex) {
    ErrorResponse errorResponse = new ErrorResponse(ex.getMessage(), LocalDateTime.now());
    return new ResponseEntity<>(errorResponse, HttpStatus.NOT_FOUND);
}
```

---

## Part 6: Validation

Try creating a new customer with an empty name or an invalid email address. The data gets saved without any complaints — this is a problem.

We cannot rely solely on the frontend to validate data. We should validate it on the backend too to prevent invalid data from reaching the database.

Add the validation dependency to `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

Add validation constraints to the `Customer` class:

```java
@NotBlank(message = "First name is mandatory")
private String firstName;

@Email(message = "Email should be valid")
private String email;
```

Add the `@Valid` annotation to the controller method:

```java
@PostMapping
public ResponseEntity<Customer> createCustomer(@RequestBody @Valid Customer customer) {
    Customer createdCustomer = customerService.createCustomer(customer);
    return new ResponseEntity<>(createdCustomer, HttpStatus.CREATED);
}
```

> **Note:** `@Valid` and `@RequestBody` can appear in either order — Spring processes them by type, not position. Both `@Valid @RequestBody` and `@RequestBody @Valid` work identically.

Test by submitting an invalid request. The validation exception will be caught by the general exception handler for now.

### Catching Validation Exceptions

We can add a dedicated handler for validation exceptions in our `GlobalExceptionHandler` to return specific, helpful error messages:

```java
import java.util.List;
import org.springframework.validation.ObjectError;
import org.springframework.web.bind.MethodArgumentNotValidException;

@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidationExceptions(MethodArgumentNotValidException ex) {

    // Get all validation errors
    List<ObjectError> validationErrors = ex.getBindingResult().getAllErrors();

    // Build an error message string from all errors
    StringBuilder sb = new StringBuilder();
    for (ObjectError error : validationErrors) {
        sb.append(error.getDefaultMessage()).append(". ");
    }

    ErrorResponse errorResponse = new ErrorResponse(sb.toString(), LocalDateTime.now());
    return new ResponseEntity<>(errorResponse, HttpStatus.BAD_REQUEST);
}
```

> **Imports needed:**
> - `org.springframework.validation.ObjectError`
> - `org.springframework.web.bind.MethodArgumentNotValidException`
>
> Your IDE should resolve these automatically. Check for red squiggles if anything fails to compile.

> **Note:** `StringBuilder` is a mutable sequence of characters, more efficient than `String` when appending multiple values. Read more [here](https://medium.com/@AlexanderObregon/understanding-string-vs-stringbuilder-in-java-50448cbbf253).

Test the validation — you should now get a structured error response with the specific validation message.

### 👨‍💻 Activity **(10 minutes)**

Add validation constraints for the following:

- Customer `yearOfBirth` should be between 1940 and 2005
- Customer `contactNo` should be exactly 8 characters long
- Interaction `remarks` should be at least 3 and at most 30 characters long
- `interactionDate` should not be in the future — use `@PastOrPresent` for this

You will need to update the `DataLoader` and any constructors accordingly.

Validation annotation references:
- https://www.baeldung.com/java-validation
- https://education.launchcode.org/java-web-development/chapters/spring-model-validation/validation-annotations.html

---

END