---
layout: post
title:  "The magic of @Transactional and its performance"
categories: spring, mysql, transactional, benchmark, spring data, jdbc
---
I was planning to write a small article about the `rollbackFor` parameter for @Trnsactional annotation. But when I 
was preparing the code examples, I decided to do some benchmarks. Let's start with the brief annotation description. 

## @Transactional

@Transactinal is annotation which allows you to work with databases' transactions in the declarative way.

{% highlight java %}
@Transactional
public void operation() {
    ... //Code which works with database
}
{% endhighlight %}

This is pretty convenient - you don't need to think about direct transactions management and exceptions handling.
Everything will be done automatically in the proxy class. This image 
(from [spring documentation](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/transaction.html)) show how class 
hierarchy looks like:

<p>
<img src="http://docs.spring.io/spring/docs/current/spring-framework-reference/html/images/tx.png" />
</p>

So, it looks pretty clear. [JavaDoc](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/transaction/annotation/Transactional.html) 
about this annotation is always there and gives enough information to get started. But there's one unclear thing with
exception handling - here's an example:

{% highlight java %}
@Transactional
public void operation() {
    entityManager.persist(new User("Dima"));
    throw new Exception();
}
{% endhighlight %}

Will it be commited? - Yes, it will! 

<p>
<img width="300" src="http://vignette2.wikia.nocookie.net/epicrapbattlesofhistory/images/5/59/Stare-What-GIF.gif/revision/latest?cb=20140928165911"/>
</p>

The documentation says:

>Although EJB container default behavior automatically rolls back the transaction on a system exception 
(usually a runtime exception), EJB CMT does not roll back the transaction automatically on anapplication 
exception (that is, a checked exception other than java.rmi.RemoteException). While the Spring default 
behavior for declarative transaction management follows EJB convention (roll back is automatic only on 
unchecked exceptions), it is often useful to customize this behavior.
>
> [Spring Documentation](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/transaction.html#transaction-declarative)

Okaaay...
So, if you expect that checked exception will be thrown in your code, you better use `rollbackFor` in this way:

{% highlight java %}
@Transactional(rollbackFor = Exception.class)
public void operation() {
    entityManager.persist(new User("Dima"));
    throw new Exception();
}
{% endhighlight %}

## Benchmarking

Now it's clear. Let's check how much we pay for the AOP magic. 
I wrote 4 simple methods to compare performance of **insert** operations using different ways to manage transactions and ORM layers:

{% highlight java %}
@Service
public class UserService {

    @Autowired
    PlatformTransactionManager transactionManager;

    @Autowired
    UserRepository repository;

    @Autowired
    @Qualifier("count")
    Integer count;

    @Autowired
    EntityManagerFactory entityManagerFactory;

    @Autowired
    DataSource dataSource;

    @Transactional
    public void transactional() {
        IntStream.range(0, count).forEach(i -> repository.save(new User("name")));
    }

    public void transactionManager() {
        DefaultTransactionDefinition def = new DefaultTransactionDefinition();
        def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        TransactionStatus transactionStatus = null;
        try {
            transactionStatus = transactionManager.getTransaction(def);
            IntStream.range(0, count).forEach(i -> repository.save(new User("name")));
            transactionManager.commit(transactionStatus);
        } catch (Exception e) {
            if (transactionStatus != null) {
                transactionManager.rollback(transactionStatus);
            }
            throw new RuntimeException(e);
        }
    }

    public void jdbc() {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            try (PreparedStatement preparedStatement = connection.prepareStatement("INSERT INTO User (name) VALUES (?)")) {

                IntStream.range(0, count).forEach(i -> {
                    try {
                        preparedStatement.setString(1, "name");
                    } catch (SQLException e) {
                        throw new RuntimeException(e);
                    }
                });
                connection.commit();
            } catch (Exception e1) {
                connection.rollback();
                throw new RuntimeException(e1);
            }
        } catch (Exception e1) {
            throw new RuntimeException(e1);
        }
    }

    public void entityManager() {
        EntityManager entityManager = entityManagerFactory.createEntityManager();
        try {
            entityManager.getTransaction().begin();
            IntStream.range(0, count).forEach(i -> entityManager.persist(new User("name")));
            entityManager.getTransaction().commit();
        } catch (Exception e) {
            entityManager.getTransaction().rollback();
        } finally {
            entityManager.close();
        }
    }
}
{% endhighlight %}

It runs in Spring Boot project with default configuration. You can check the source code of this benchmark out on [github](https://github.com/dimafeng/dimafeng-examples) (`gradlew transactional:run`). That's what I've got:

<p>
<img src="{{ site.url }}/assets/transactional.png" class="img-responsive">
</p>

This chart shows the best execution time from 20 attempts for 100, 1000, and 10000 inserts. As you see, **@Transactional + Spring Data** works almost the same as working with **EntityManager** directly. For 10000 inserts, the time difference between **@Transactional** and **EntityManager** was 250ms - it's 4% of total execution time, and it's much less than the difference between pure **JDBC** and **EntitiyManager**.  