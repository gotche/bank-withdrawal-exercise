# Bank Withdrawal Exercise

## Introduction
This is the solution to the proposed exercise as part of the application for a tech lead role.
My main focus will be the analysis of the problem and possible solutions. This approach brings the most value for the exercise and allows the reader to evaluate my technical competence. 
It also means I will leave out most of the implementation effort since I don’t expect testing my limited knowledge of the Java language to be the goal of the exercise.

## Approach
I have divided my approach into 3 parts:
1. Understanding business goals 
2. Prioritize which dimensions to look at, based on business goals. 
3. Evaluate the code regarding the prioritized dimensions and propose solutions.


### Understanding Business Goals

The available code is mostly an api endpoint that follows this pseudocode:

- post endpoint receives accountId and amount.
- retrieves the current balance of the provided accountId from the database
- Checks if there is enough balance in the account to withdraw the requested amount
- updates the row corresponding to the accountId with the new balance (new balance = old balance - amount)
- an event is created and fanout via AWS SNS

As a bank business, I am assuming operational correctness and Data Integrity is paramount. We will design for Consistency over Availability. 

I will also assume the event creation is important, but we will neither roll back the withdrawal transaction nor fail the request if the event creation fails. We will need to find a mechanism to ensure the event is created even if it is a bit delayed.

Lastly, it is mentioned that business is expanding, so scalability will be important, but also maintainability and testability. As the business grows and expands, we should not only look at core functionality but also be able to add new features quickly and with confidence.

### Prioritize Dimensions
The exercise enumerates the following dimensions:
structure, throughput, maintainability, scalability, flexibility, consistency, fault tolerance, testability, dependency management, observability, auditability, portability, correctness, cost efficiency and overall quality

Looking at business impact, my prioritized list will be:

#### Correctness 
For customers, banking is all about trust. Buggy software reduces trust, apart from more direct economic consequences.
#### Consistency 
Again, not having consistent data or behavior will result in economic loss.
#### Auditability
This is probably a compliance requirement, but as a customer, I would expect a bank to be able to understand how my money is processed.
#### Observability 
Understanding how the system behaves is key to guaranteeing the first 3 points
#### Fault tolerance 
Having some tolerance for hardware failure is also important to maintaining a good customer experience
#### Scalability 
As per the exercise statement, the system is poised to expand, so we need to make sure we can serve all customers plus expected growth plus a margin.
#### Maintainability and testability 
Code well maintained that can be tested contains fewer bugs and can be improved faster.
#### Throughput and Cost efficiency 
The higher the throughput the larger the number of customers can be served in parallel. Usually, throughput and cost efficiency are on opposite parts of the spectrum, and tradeoffs will be necessary.
#### Portability 
Not sure how this is important
About the code; the JVM has been ported to a large number of architectures. 
I am not concerned about SQL portability either, since it is unlikely a problem in reality. And if there is the need to migrate databases we will approach it as a migration.
#### Flexibility, Structure, and dependency management
I believe these last 3 dimensions depend tremendously on the language the application is expressed and I have no experience writing Java applications. 
Don't get me wrong, having a good structure and flexible code would impact delivery speed. I will not propose any improvements related to them because I am not fluent in the specific Java idioms.

### Evaluate the code and propose solutions
#### Correctness
- The elephant in the room is the race condition happening while reading from the database and then using those 'potentially outdated' values to update the balance. There is a possibility that 2 concurrent calls within the same account violate the balance check and leave the account in negative numbers. 

```java
       // Check current balance
        String sql = "SELECT balance FROM accounts WHERE id = ?";
        BigDecimal currentBalance = jdbcTemplate.queryForObject(
        sql, new Object[]{accountId}, BigDecimal.class);
        if (currentBalance != null && currentBalance.compareTo(amount) >= 0) {
            // Update balance
            sql = "UPDATE accounts SET balance = balance - ? WHERE id = ?";
            int rowsAffected = jdbcTemplate.update(sql, amount, accountId);
            if (rowsAffected > 0) {
                return "Withdrawal successful";
            } else {
            // In case the update fails for reasons other than a balance check
                return "Withdrawal failed";
        }

```
- The default isolation level used in JDBC is “read-committed” which allows the race condition to happen. We could solve this situation using higher transaction isolation guarantees, like Serializable transactions. The price to pay would be a lower throughput since the database will be effectively serializing transactions.

```java
      Connection connection = null;
      private static final String DB_URL = "jdbc:mysql://localhost:3306/mydatabase";
      private static final String USER = "yourusername";
      private static final String PASSWORD = "yourpassword";

        try {
            // Establish the connection
            connection = DriverManager.getConnection(DB_URL, USER, PASSWORD);
            // Set isolation level to Serializable
            connection.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
            // Disable auto-commit mode
            connection.setAutoCommit(false);

            // First SQL statement - SELECT query
            String sql1 = "SELECT balance FROM accounts WHERE id = ?";
            try (PreparedStatement pstmt1 = connection.prepareStatement(sql1)) {
                pstmt1.setDouble(1, accountId);
                try (ResultSet rs = pstmt1.executeQuery()) {
                     rs.next()
                      BigDecimal balance = rs.getBigDecimal("balance");
                                            
                }
            }
            if (currentBalance != null && currentBalance.compareTo(amount) >= 0) {
                // Second SQL statement - Update query
                String sql2 = "UPDATE accounts SET balance = balance - ? WHERE id = ?";
                try (PreparedStatement pstmt2 = connection.prepareStatement(sql2)) {
                    pstmt2.setString(1, amount);
                    pstmt2.setString(2, accountId);
                    pstmt2.executeUpdate();
                }
           }
            else {
                throw new OffLimitsException(“Not enough balance to perform withdrawal”);
           }
            // Commit the transaction
            connection.commit();
            System.out.println("Transaction committed successfully.");

        } catch (SQLException | OffLimitsException e) {
            // Handle the exception and roll back the transaction
            if (connection != null) {
                try {
                    System.out.println("Rolling back the transaction.");
                    connection.rollback();
                } catch (SQLException ex) {
                    ex.printStackTrace();
                }
            }
            e.printStackTrace();
        } finally {
            // Close the connection
            if (connection != null) {
                try {
                    connection.setAutoCommit(true); // Reset to default state
                    connection.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

- There are other problems like not checking the amount to withdraw is a positive number, otherwise it would not be a withdrawal. Or allowing any quantity to be withdrawn (which is usually restricted by a maximum withdrawal value) 

- There is no idempotency check. (assuming no race condition) Getting several identical requests in a short period is usually unintended. We can solve this situation using an idempotency key per transaction, sent from the client/frontend and persisted in the database.

- The code of publishing the event to the SNS topic is never reached since the function returns in all the cases (success, failure, withdrawal off limits). This can be solved by deleting the return success and letting the program flow reach the end of the function.

- The publish call to SNS could fail (AWS offers a 99.9% availability SLA) so we can introduce a retry mechanism for that request. The retries could also fail, in this case I would emit a log and let the withdrawal request finish successfully. We could have a ‘repairing job’ outside of the request, that would scan the transactions table (this table will be introduced in the next section) for missing withdrawal events, and emit a new one in case of a detected miss. 

#### Auditability
- The next important fact is that there is no Auditability, the only data stored in the database is the current balance amount (just a point in time, the present). There are no details or history of different withdrawal transactions. To solve this I would add a new append-only table called transactions, where the transaction details would be stored (date, amount, account_id, previous_balance). Appending the new row must happen within the same database transaction.

- We could even double down here and introduce a double-entry ledger, to guarantee data consistency from the accounting perspective.

#### Observability
- There are no logs, traces, or metrics exposed. 
- The code is not instrumented. Observability is key to understanding the health and performance of the application in production.
- For this functionality, I would use logging and log all the paths that lead to a non-successful operation
  ```java
  // after the check of amount > currentBalance
  logger.info("Withdrawal aborted: not enough balance");
  ...
  // after retrying SNS request
  logger.error("SNS Message Withdrawal Event was not sent");
  ```
- I would propose structured logging. Not just a message but adding extra context is important. Logging the hashed accountId, for example, could benefit anti-fraud analysis. 
- Lastly, there are tools involved to be able to use logs/traces/metrics effectively (debug confidently, get alerted quickly, monitor SLOs...). Using a hosted solution based on open-source technology (something like ELK) could be advisable. Other tools like Sentry aggregate errors and allow the development team to prioritize more impactful bugs.

#### Fault Tolerance
- Currently, if the database or SNS service has a temporary unavailability our application would fail. Adding retries (configuring proper timeouts) would make the app more fault-tolerant.

- The application should work behind a load balancer with failure detection capabilities.
- If we look also at the system level, we need to set up database replication mechanisms to promote a replica to leader in case of failure.

- Finally, let’s ensure we have database backups (they are performed and tested regularly) and a disaster recovery strategy.

#### Scalability
- First of all, we need to measure what the current baseline is. How many withdrawal transactions can the application support per second? We can do this through Load Testing, with tools like Locust. We also need to run an analysis to determine what the scaling target would be (a typical value could be around 2X-3X). 

- At the application level, we could benefit from a database connection pool, since starting a new database connection is one of the most expensive operations.

- If we hit a database bottleneck, we could scale the instance vertically, as a first step and then move to a bit more complex architecture, via horizontal scaling with different partitions or shards using a partition key based on the accountId (consistent hashing). This new distributed architecture requires the implementation of distributed transactions which also comes at a price. 

- Another improvement could be extracting the SNS message functionality and running it async or as a background job. Or even removing it completely by using the Change Data Capture pattern on the transactions table.

#### Maintainability 
- There are a few examples of config values that show up in the middle of the code:
  ```java
  snsTopicArn = "arn:aws:sns:YOUR_REGION:YOUR_ACCOUNT_ID:YOUR_TOPIC_NAME";
  ```
  They should be put under environment-dependent configuration, so we could have different values for production, staging, development, testing, and local.
- Instead of returning strings as a result of the post request api ("Withdrawal successful"), I would return a dictionary structure with actionable information. Another improvement would be defining these strings in a class or enum, so it would become less error-prone when developers use them in the code.
  ```
  {
    "error": "balance-0001",
    "message": "Unsufficient Balance",
    "detail": "The bank doesn't allow to withdraw a higher amount than the current balance in this account",
    "help": "https://example.com/help/error/balance-0001"
  }
  ```
- HTTP error codes are missing as well. If the code hits any of the conditionals that check the balance, I would not return HTTP 200 but HTTP 400 Bad Request

- The flow of the conditionals can be inverted so failing conditions will execute first and will provide a quick exit of the function.
  ```java
  if (currentBalance == null) {
     return NO_BALANCE_FOUND }
  if (currentBalance.compareTo(amount) < 0){
     return NOT_ENOUGH_BALANCE }
  if (amount < 0) {
     return CANT_WITHDRAW_NEGATIVE_AMOUNT }
  ...
  ```

#### Testability
- The code of the controller is not easily testable.
- There are external calls to the DB and SNS. We could use dependency injection to improve testability.
- Extracting the logic of the SNS message would also allow us to unit test it in isolation.
- Same for the new balance logic. They could be extracted into a pure function and be unit tested.


## References

- [1] https://www.postgresql.org/docs/current/transaction-iso.html
- [2] Designing Data-Intensive Applications
- [3] https://aws.amazon.com/messaging/sla/
- [4] chatgpt question: when publishing a message to AWS SNS from java code, can you give me an example of logic to retry publishing in case the initial publish call fails?
- [5] https://rollbar.com/guides/java/how-to-throw-exceptions-in-java/
- [6] https://www.marcobehler.com/guides/java-logging
