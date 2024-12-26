---
layout: post
title: "Managing Database Transactions in Clean Architecture: The Hidden Complexities"
date: 2024-12-26 00:00:00 +0000
excerpt: "Database transactions appear deceptively simple but contain hidden complexities that can compromise data consistency. Through a practical example of user registration, we'll explore common pitfalls and set the stage for robust transaction management patterns."
---

Database transactions seem simple at first glance. Create a record, update some fields, commit the changes - what could
go wrong? Yet as our applications grow more complex, maintaining data consistency becomes increasingly challenging,
especially when multiple operations need to succeed or fail together.

Over years of building enterprise applications, I've observed teams repeatedly struggle with transaction management.
The challenges often emerge subtly: an account created without its required user records, notifications sent for
failed operations, or data left in inconsistent states after partial failures. These issues typically stem not from
obvious bugs, but from misconceptions about transaction boundaries and responsibilities.

In this series, we'll explore how to implement robust transaction management using [SQLAlchemy] and [FastAPI] while 
adhering to [clean architecture] principles. We'll progress from common pitfalls through increasingly sophisticated 
solutions, developing patterns that not only solve immediate problems but scale well as applications grow in complexity.

## The Challenge: A Simple User Registration System

Let's start with a seemingly straightforward example that reveals fundamental challenges in transaction management. 
Consider a user registration system where creating a new account requires both an account record and associated user 
information. Many teams begin with repository-level transactions, where each repository handles its own database operations:

```python
class AccountRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def save(self, entity: Account) -> None:
        model = self.to_model(entity)
        self._session.add(model)
        self._session.commit()  # Repository handles its own transaction
        
class UserRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def save(self, entity: User) -> None:
        model = self.to_model(entity)
        self._session.add(model)
        self._session.commit()  # Repository handles its own transaction
```

The use case orchestrates these repositories to create both records:

```python
class SignUpUseCase(UseCase[CreateAccountCommand, tuple[Account, User]]):
    def _execute(self, command: CreateAccountCommand) -> tuple[Account, User]:
        # Hash password before any database operations
        password_hash = self._password_hashing_service.hash(command.password)
        
        # Domain logic ensures user and account are created together
        user, account = User.create_with_account(command.email, password_hash)

        with self._session_factory() as session:           
            user_repository = UserRepository(session)
            account_repository = AccountRepository(session)

            # These operations need to be atomic but currently aren't
            account_repository.save(account)  # First transaction
            user_repository.save(user)        # Second transaction

            return account, user
```

This implementation follows clean architecture principles - each repository encapsulates its database operations, and 
the use case coordinates the business process. However, it contains a subtle but critical flaw: we're treating the 
creation of an account and its admin user as separate transactions.

Consider this sequence:
1. The account creation succeeds and commits
2. The user creation fails (perhaps due to a duplicate email)
3. We're left with an account without an admin user - a violation of our domain rules

<figure>
    <img src="/assets/2024-12-26/separate-transactions.png"/>
    <figcaption>Figure 1 - Separate Transactions</figcaption>
</figure>

Here's a test that exposes this issue:

```python
def test_demonstrates_partial_commit_problem(use_case: SignUpUseCase, session_factory: sessionmaker[Session]) -> None:
    command = CreateAccountCommand(
        email="john@example.com",
        password="password",
        password_confirmation="password"
    )

    use_case(command)  # First creation succeeds

    with pytest.raises(IntegrityError):  # Second attempt fails
        use_case(command)

    with session_factory() as session:
        orphaned_accounts = (
            session.execute(
                select(AccountModel)
                .outerjoin(UserModel)
                .where(UserModel.uid.is_(None))
            ).scalars().all()
        )

        assert len(orphaned_accounts) > 0  # We have orphaned accounts!
```

This pattern reveals a broader truth about transaction management: what seems simple in isolation becomes complex when 
we consider the full business context. Each repository working independently makes perfect sense from a code 
organization perspective. Yet this independence creates subtle but dangerous gaps in our data consistency.

## Setting the Stage for Better Solutions

The root issue runs deeper than just implementation details. When we follow clean architecture principles, we're taught 
to keep our components focused and independent. Each repository managing its own transactions seems to align perfectly 
with this philosophy - it's encapsulated, testable, and follows single responsibility. Yet this independence creates a 
subtle but dangerous gap between our technical implementation and business reality.

Creating an account with its admin user isn't just two separate operations that happen to occur together - it 
represents a single, atomic business operation. From our domain's perspective, an account cannot exist without an 
admin user. By splitting this operation across multiple transactions, we've inadvertently prioritized technical 
separation over business rules, opening the door to data inconsistencies that violate our domain invariants.

This tension between clean architecture principles and transactional requirements reveals several essential questions 
that we need to address:

First, how do we maintain clean separation of concerns while ensuring data consistency? Our repositories should remain 
focused and independent, but they also need to participate in larger transactional contexts. We need patterns that 
allow components to be both independent and coordinated.

Second, where should transaction boundaries live in a clean architecture? The traditional layers - entities, use cases, 
interfaces, and infrastructure - don't explicitly address transaction management. Should it be a cross-cutting concern? 
Part of the use case layer? We need to find a natural home for these boundaries that doesn't violate our architectural 
principles.

Third, how do we handle complex operations that span both critical and optional steps? Real-world business operations 
often include a mix of must-succeed database operations and nice-to-have side effects like sending notifications or 
updating caches. We need patterns that can distinguish between these different types of operations and handle their 
failures appropriately.

Throughout this series, we'll explore these questions through increasingly sophisticated solutions. We'll begin with 
the Unit of Work pattern as our foundation, showing how it helps coordinate multiple repositories while maintaining 
clean architectural boundaries. From there, we'll progress to more advanced topics including managing complex business 
operations, handling nested transactions effectively, and implementing these patterns in production systems. 

Each article will include practical Python implementations using SQLAlchemy and FastAPI, allowing you to follow along 
and adapt these patterns to your own applications. We'll maintain our focus on clean architecture principles while 
developing solutions that scale with your application's complexity.

[//]: ---

[//]: # (*This is part 1 of a series on Clean Architecture Transaction Management. [Part 2] explores the Unit of Work pattern ) as a solution to these challenges.*

[Part 2]: #part-2
[SQLAlchemy]: https://www.sqlalchemy.org
[FastAPI]: https://fastapi.tiangolo.com
[clean architecture]: https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html

