# Self Studies: Persistent Data Management and Robust API Design

## Overview

In this lesson you will switch `simple-crm` from H2 to PostgreSQL and make the API production-ready with global exception handling and input validation. The self-study materials below focus on the most conceptually new content — exception handling and validation — so you can follow the code-along confidently. PostgreSQL setup and derived queries will be walked through step by step in class.

**Estimated Prep Time:** 60–80 minutes

---

## Task 1: Spring Boot Exception Handling and Validation

This video covers how to build a global exception handler using `@ControllerAdvice`, return structured error responses, and apply validation annotations to your Spring Boot application. This maps directly to Parts 4, 5, and 6 of the lesson.

**Watch:** Spring Boot Exception Handling and Validation Tutorial
🎬 https://www.youtube.com/watch?v=2o7LJLTIgdE

**Then read:** Lesson 3.17 — Parts 4 to 6

**Guiding Questions:**
- What is the problem with calling `.get()` directly on an `Optional`?
- What does `@ControllerAdvice` do and why is it better than `try-catch` blocks in every controller?
- What is the purpose of the `ErrorResponse` class?
- What does `@Valid` do and where must it be placed relative to `@RequestBody`?

---

## Task 2: Read — PostgreSQL Setup and Derived Queries

No video needed for this — read the lesson sections directly before class so the setup steps feel familiar.

**Read:** Lesson 3.17 — Parts 1 to 3

**Guiding Questions:**
- What is the main difference between H2 and PostgreSQL in terms of persistence?
- Which properties in `application.properties` need to change when switching from H2 to PostgreSQL?
- How does Spring JPA know what SQL to generate from a method name like `findByFirstName`?

---

## Active Engagement Strategies

- Before class, make sure PostgreSQL is installed and running on your machine — the first 20 minutes of the lesson depend on it. Test the connection using DBeaver
- After watching the video, try to write the `GlobalExceptionHandler` class from memory — just the skeleton with `@ControllerAdvice` and one `@ExceptionHandler` method
- Think about the `simple-crm` project — what invalid data could a user currently submit? Make a list and bring it to class

---

## Additional Reading Material

- [Spring Boot Exception Handling — Baeldung](https://www.baeldung.com/exception-handling-for-rest-with-spring)
- [Java Optional — Baeldung](https://www.baeldung.com/java-optional)
- [Spring Data Derived Queries — Baeldung](https://www.baeldung.com/spring-data-derived-queries)
- [Java Bean Validation Annotations — Baeldung](https://www.baeldung.com/java-validation)