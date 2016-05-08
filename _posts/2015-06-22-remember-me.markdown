---
layout: post
title:  "Remember me with Spring framework and Mongo DB"
categories: java, spring mvc, mongo db, spring security
---
Today I'm going to show you how to impelement remember-me functionality for spring mvc. I'll use persistant tokens which
will be stored in mongodb (I work with mongo via spring data). This is useful approach when you start your application within a cluster. If one server is
down all clients will be redirected to another server, and this server can validate cookies using data of first server
from database. If I used some of Relational database we whould use spring security built-in classes. Unfortunately, spring
dosn't have such built-in solutions for mongo, but it has a lot of interfaces wich we can quickly implement and reach the
same goal with almost the same effort.

First of all, we have to create document for tokens storing:

{% highlight java %}
@Document
@CompoundIndexes({
        @CompoundIndex(name = "i_username", def = "{'username': 1}"),
        @CompoundIndex(name = "i_series", def = "{'series': 1}")
})
public class Token extends PersistentRememberMeToken {

    @Id
    private final String id;

    @PersistenceConstructor
    public Token(String id, String username, String series, String tokenValue, Date date) {
        super(username, series, tokenValue, date);
        this.id = id;
    }

    public String getId() {
        return id;
    }
}
{% endhighlight %}

`PersistentRememberMeToken` is basic class for token, it contains all required fields such as `username`,
`series`, `tokenValue`, `date`. All fields declared as final, that's why we have to use `@PersistenceConstructor` annotaion
otherwise a class should have default constructor. Another tricky appoach is the using @CompoundIndexes annotation to
specify indexes, it's not oblivius way to add indexes, but we have to use it due to the fact that we cannot add
annotations to parent class.

Next we have to describe repository for spring data:

{% highlight java %}
public interface TokenRepository extends MongoRepository<Token, String> {
    Token findBySeries(String series);
    Token findByUsername(String username);
}
{% endhighlight %}

As you see it's 4 lines of code, and this code is simple. Btw, if you don't know what spring data is, these 2 classes are all what you need to work with a collection in mongo db.

Now we can implement `PersistentTokenRepository`. This class isn't Repository in terms of spring data, it's interface
from spring security and defines four methods which helps manage and verify tokens. It could be implemented as follows:

{% highlight java %}
@Component
public class TokenService implements PersistentTokenRepository {

    @Autowired
    TokenRepository repository;

    @Override
    public void createNewToken(PersistentRememberMeToken token) {
        repository.save(new Token(null,
                token.getUsername(),
                token.getSeries(),
                token.getTokenValue(),
                token.getDate()));
    }

    @Override
    public void updateToken(String series, String value, Date lastUsed) {
        Token token = repository.findBySeries(series);
        repository.save(new Token(token.getId(), token.getUsername(), series, value, lastUsed));
    }

    @Override
    public PersistentRememberMeToken getTokenForSeries(String seriesId) {
        return repository.findBySeries(seriesId);
    }

    @Override
    public void removeUserTokens(String username) {
        Token token = repository.findByUsername(username);
        if (token != null) {
            repository.delete(token);
        }
    }
}
{% endhighlight %}

As you see, all operations with tokens go through our spring data repository. I think this code dosn't require any comments.

The last step is a connection of our repository with spring security config:

{% highlight java %}

@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(securedEnabled = true)
public class AppSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    PersistentTokenRepository repository;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
		...
        .and()
        .rememberMe()
        .tokenRepository(repository);
    }

    ...
}
{% endhighlight %}

This code allows configuring of Remember Me authentication. `tokenRepositor` methos takes our `repository` as a parameter. Also you can adjust some additional parameters, e.g. you can change value of `tokenValiditySeconds`.

The working example you can find in my [pet project](https://github.com/dimafeng/cards/commit/6672abfa97636974162f48cc0a9e0c908c6e2a70)
