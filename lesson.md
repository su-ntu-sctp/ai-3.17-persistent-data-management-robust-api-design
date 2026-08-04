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

Comment out the H2 dependencies. Since Lesson 3.16, your `pom.xml` has **two** H2-related dependencies — the H2 database driver itself, and the H2 console starter that was added so the console works on Spring Boot 4's modularised auto-configuration. Both need to be commented out when switching to Postgres:

```xml
<!-- <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
</dependency> -->

<!-- <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-h2console</artifactId>
</dependency> -->
```

> **Note:** If you only comment out `h2database:h2` and leave `spring-boot-h2console` in place, the app will still run fine — the H2 console auto-config simply has nothing to attach to since there's no H2 datasource. But it's cleaner to remove both together, since neither is needed once you're fully on Postgres.

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

> **Note:** Spring Boot 4.x with Hibernate 7 auto-detects the PostgreSQL dialect — you do not need to set `spring.jpa.database-platform` manually. Omitting it is the recommended approach.

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

### Searching by Multiple Fields

To search by both first name and last name, chain the fields with `And`:

```java
List<Customer> findByFirstNameAndLastName(String firstName, String lastName);
```

Spring JPA reads the method name and generates:

```sql
SELECT * FROM customer WHERE first_name = ? AND last_name = ?
```

> **Note:** The field names in the method name must exactly match the field names in your entity class. So if your entity has `firstName` (camelCase), the method uses `FirstName` (capitalised camelCase after `findBy`). Spring handles the translation to `first_name` in SQL automatically.

Since `/search` is already taken by `findByFirstName`, use a different path for this endpoint:

```java
@GetMapping("/search/full")
public ResponseEntity<List<Customer>> searchCustomers(
        @RequestParam String firstName,
        @RequestParam String lastName) {
    List<Customer> foundCustomers = customerService.searchCustomers(firstName, lastName);
    return new ResponseEntity<>(foundCustomers, HttpStatus.OK);
}
```

Test it:

```
http://localhost:8080/customers/search/full?firstName=Stephen&lastName=Strange
```

For more information, read about [JPA Derived Query from Method Name](https://www.baeldung.com/spring-data-derived-queries).

---

## Part 4: `Optional` and Exception Handling

In Lesson 3.16, you already refactored `CustomerServiceImpl` to use `.orElseThrow()` with a generic `RuntimeException`:

```java
@Override
public Customer getCustomer(Long id) {
    return customerRepository.findById(id)
        .orElseThrow(() -> new RuntimeException("Customer not found with id: " + id));
}
```

This is a good start — `.orElseThrow()` is the correct approach over `.get()`. However, throwing a generic `RuntimeException` makes it impossible to handle this specific case cleanly. The next step is to replace it with a **custom exception class**.

### Why a Custom Exception?

With a generic `RuntimeException`, our `GlobalExceptionHandler` (coming in Part 5) cannot target it specifically — it would catch unrelated exceptions too. A dedicated `CustomerNotFoundException` lets us:
- Return a meaningful `404 NOT FOUND` response specifically for this case
- Keep other unexpected exceptions returning `500 INTERNAL SERVER ERROR`

### Create `CustomerNotFoundException`

Create a new class `CustomerNotFoundException.java`:

```java
public class CustomerNotFoundException extends RuntimeException {
    public CustomerNotFoundException(Long id) {
        super("Customer not found with id: " + id);
    }
}
```

> **Note:** The constructor takes a `Long id` parameter — not a `String`. This matches the `id` type used throughout the application.

### Update `CustomerServiceImpl`

Now update all methods that do a customer lookup to throw `CustomerNotFoundException` instead of `RuntimeException`:

```java
@Override
public Customer getCustomer(Long id) {
    return customerRepository.findById(id)
        .orElseThrow(() -> new CustomerNotFoundException(id));
}

@Override
public Customer updateCustomer(Long id, Customer customer) {
    Customer customerToUpdate = customerRepository.findById(id)
        .orElseThrow(() -> new CustomerNotFoundException(id));
    customerToUpdate.setFirstName(customer.getFirstName());
    customerToUpdate.setLastName(customer.getLastName());
    customerToUpdate.setEmail(customer.getEmail());
    customerToUpdate.setContactNo(customer.getContactNo());
    customerToUpdate.setJobTitle(customer.getJobTitle());
    customerToUpdate.setYearOfBirth(customer.getYearOfBirth());
    return customerRepository.save(customerToUpdate);
}

@Override
public void deleteCustomer(Long id) {
    customerRepository.findById(id)
        .orElseThrow(() -> new CustomerNotFoundException(id));
    customerRepository.deleteById(id);
}

@Override
public Interaction addInteractionToCustomer(Long id, Interaction interaction) {
    Customer selectedCustomer = customerRepository.findById(id)
        .orElseThrow(() -> new CustomerNotFoundException(id));
    interaction.setCustomer(selectedCustomer);
    return interactionRepository.save(interaction);
}
```

> **Note:** `deleteCustomer` now does a lookup before deleting. This ensures the client gets a meaningful `404` if they try to delete a non-existent customer, rather than a silent no-op.

---

## Part 5: Global Exception Handler

Spring Boot lets us create a global exception handler using the `@RestControllerAdvice` annotation. This gives us a single centralised place to handle exceptions thrown from any controller in the application — no more scattered `try-catch` blocks.

### Why `@RestControllerAdvice`?

Without a global handler, every controller method that could throw an exception needs its own `try-catch` block. This leads to repetitive, cluttered code. With `@RestControllerAdvice`, exceptions bubble up from the service layer, through the controller, and are caught in one place automatically.

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

With this in place, we can remove the `try-catch` blocks from our controller. Since `@RestControllerAdvice` intercepts exceptions from **any** controller method in the application, you can remove `try-catch` from all methods that throw `CustomerNotFoundException` — that includes `getCustomer`, `updateCustomer`, and `deleteCustomer`. Only `createCustomer` is unaffected since it does not do a lookup by ID.

```java
@GetMapping("/{id}")
public ResponseEntity<Customer> getCustomer(@PathVariable Long id) {
    Customer foundCustomer = customerService.getCustomer(id);
    return new ResponseEntity<>(foundCustomer, HttpStatus.OK);
}
```

When `CustomerNotFoundException` is thrown anywhere in the application, it will be caught and handled by the global handler automatically.

### Structured Error Response

Currently we are returning the error as a plain string. This works, but it is not ideal — the frontend receives an inconsistent response (sometimes a JSON object, sometimes a plain string) which makes it harder to parse and display errors meaningfully.

We can fix this by creating a dedicated `ErrorResponse` class that gives every error a consistent structure with a message and a timestamp:

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

Now every error response from this handler looks like this:

```json
{
    "message": "Customer not found with id: 99",
    "timestamp": "2026-06-18T15:45:36.189"
}
```

The frontend always knows what shape to expect, making error handling predictable and reliable.

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

A few things to note here:
- **`Exception.class`** — this is the base class of all exceptions in Java, so this handler catches anything that has not been explicitly handled by a more specific `@ExceptionHandler` method above it.
- **`HttpStatus.INTERNAL_SERVER_ERROR` (500)** — we return `500` because an unhandled exception means something unexpected happened on the **server side**, not because the client sent a bad request. `500` tells the client "this is our problem, not yours."
- **`"Something went wrong"`** — we deliberately return a generic message instead of the real exception message. This is a security best practice — internal error details (stack traces, database errors, class names) should never be exposed to the client as they can reveal implementation details that attackers could exploit.

### 👨‍💻 Activity **(15 minutes)**

**Step 1** — Create a new `InteractionNotFoundException` class that extends `RuntimeException`:

```java
public class InteractionNotFoundException extends RuntimeException {
    public InteractionNotFoundException(Long id) {
        super("Interaction not found with id: " + id);
    }
}
```

**Step 2** — Update `GlobalExceptionHandler` to handle both `CustomerNotFoundException` and `InteractionNotFoundException` in a single method using array notation:

```java
@ExceptionHandler({CustomerNotFoundException.class, InteractionNotFoundException.class})
public ResponseEntity<ErrorResponse> handleResourceNotFoundException(RuntimeException ex) {
    ErrorResponse errorResponse = new ErrorResponse(ex.getMessage(), LocalDateTime.now());
    return new ResponseEntity<>(errorResponse, HttpStatus.NOT_FOUND);
}
```

Since both exceptions extend `RuntimeException`, the parameter type is `RuntimeException` and `ex.getMessage()` works for both.

> **Note:** `InteractionServiceImpl` is not part of this module — `Interaction` was introduced solely to demonstrate the many-to-one relationship. `InteractionNotFoundException` is created here for completeness and good practice, as it can be used in `CustomerServiceImpl` if interaction lookups are needed in future.

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

### Catching Validation Exceptions (Optional)

We can add a dedicated handler for validation exceptions in our `GlobalExceptionHandler` to return specific, helpful error messages.

When `@Valid` fails, Spring throws a `MethodArgumentNotValidException`. This exception contains a `BindingResult` — an object that holds all the validation errors that occurred. Each individual error is represented as an `ObjectError`, which contains the message from the validation annotation (e.g. `"First name is mandatory"`).

We loop through all the errors and build a single combined error message using a `StringBuilder` — a mutable sequence of characters that is more efficient than concatenating strings with `+` in a loop. Read more [here](https://medium.com/@AlexanderObregon/understanding-string-vs-stringbuilder-in-java-50448cbbf253).

We return `400 BAD_REQUEST` because validation failures are the **client's fault** — they sent invalid data. This is different from `500 INTERNAL_SERVER_ERROR` which means something went wrong on the server side.

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

Test the validation — you should now get a structured error response with the specific validation message.

### 👨‍💻 Activity **(10 minutes)**

Add validation constraints for the following:

- Customer `contactNo` should be exactly 8 characters long
- Interaction `remarks` should be at least 3 and at most 30 characters long
- `interactionDate` should not be in the future — use `@PastOrPresent` for this

> ⚠️ **Important:** This activity only produces the clean `400 Bad Request` response you expect if the Interaction-creation endpoint (`addInteractionToCustomer` in `CustomerController`, from Lesson 3.16 Part 5) has `@Valid` on its `@RequestBody Interaction interaction` parameter — the same way `createCustomer` does above. Without `@Valid` there, invalid Interaction data skips controller-level validation entirely and instead fails later at persist time via Hibernate's own Bean Validation, throwing `ConstraintViolationException` instead of `MethodArgumentNotValidException`. Since `handleValidationExceptions` only catches the latter, a missing `@Valid` here means the request falls through to the generic `Exception.class` handler and you get a generic `500` — "Something went wrong" — with the real validation message only visible in the console log, not in the API response. Before starting this activity, confirm `@Valid` is present:
>
> ```java
> @PostMapping("/{id}/interactions")
> public ResponseEntity<Interaction> addInteraction(
>         @PathVariable Long id,
>         @RequestBody @Valid Interaction interaction) {
>     Interaction createdInteraction = customerService.addInteractionToCustomer(id, interaction);
>     return new ResponseEntity<>(createdInteraction, HttpStatus.CREATED);
> }
> ```

Validation annotation references:
- https://www.baeldung.com/java-validation
- https://education.launchcode.org/java-web-development/chapters/spring-model-validation/validation-annotations.html

---

END