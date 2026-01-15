
# Assignment (Optional)

## Brief

Create a Spring Boot project called EmployeeManagementSystem with PostgreSQL database and implement exception handling and validation.

1. **PostgreSQL Setup with JPA Derived Queries**
   - Install PostgreSQL and create a database called "employee_db"
   - Create a Spring Boot project with dependencies: Spring Data JPA, PostgreSQL Driver, Spring Web, Lombok, and Validation
   - Configure `application.properties`:
     - Set up PostgreSQL connection (url, username, password)
     - Set `spring.jpa.hibernate.ddl-auto=create`
   - Create an `Employee` entity with:
     - `Long id` (primary key with auto-generation)
     - `String name`, `String email`, `String department`
     - Use Lombok's @Getter and @Setter
   - Create `EmployeeRepository` extending `JpaRepository<Employee, Long>`
   - Add a derived query method: `List<Employee> findByDepartment(String department)`
   - Create `EmployeeService` interface and implementation with methods:
     - `createEmployee()`, `getAllEmployees()`, `getEmployee(Long id)`, `searchByDepartment(String department)`
   - In the service implementation, use `orElseThrow()` with custom `EmployeeNotFoundException` when getting an employee
   - Create `EmployeeController` with endpoints:
     - `POST /employees` - Create an employee
     - `GET /employees` - Get all employees
     - `GET /employees/{id}` - Get an employee by ID
     - `GET /employees/search?department=IT` - Search employees by department
   - Create a `DataLoader` to pre-populate 4 employees from 2 different departments
   - Test all endpoints and verify data in PostgreSQL using DBeaver or psql

2. **Global Exception Handler and Validation**
   - Create a custom exception `EmployeeNotFoundException` that extends `RuntimeException`
   - Create an `ErrorResponse` class with fields: `String message` and `LocalDateTime timestamp` (use Lombok)
   - Create a `GlobalExceptionHandler` class with @ControllerAdvice
   - Add exception handlers:
     - `@ExceptionHandler(EmployeeNotFoundException.class)` - Return 404 with ErrorResponse
     - `@ExceptionHandler(MethodArgumentNotValidException.class)` - Handle validation errors and return 400 with ErrorResponse
     - `@ExceptionHandler(Exception.class)` - General handler returning "Something went wrong" with 500 status
   - Add validation annotations to the Employee entity:
     - `@NotBlank(message = "Name is mandatory")` on name field
     - `@Email(message = "Email should be valid")` on email field
     - `@NotBlank(message = "Department is mandatory")` on department field
   - Update the `POST /employees` endpoint to use `@Valid` before `@RequestBody`
   - Test the following scenarios:
     - Create an employee with valid data (should succeed)
     - Create an employee with blank name (should return validation error)
     - Create an employee with invalid email (should return validation error)
     - Get an employee with invalid ID (should return EmployeeNotFoundException)

## Submission (Optional)

- Submit the URL of the GitHub Repository that contains your work to NTU black board.
- Should you reference the work of your classmate(s) or online resources, give them credit by adding either the name of your classmate or URL.

## References
- Java: https://docs.oracle.com/javase/
- Spring Boot: https://docs.spring.io/spring-boot/docs/current/reference/html/
- PostgreSQL: https://www.postgresql.org/docs/
- OWASP: https://cheatsheetseries.owasp.org/