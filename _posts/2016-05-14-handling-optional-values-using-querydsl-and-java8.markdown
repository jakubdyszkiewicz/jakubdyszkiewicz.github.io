---
layout: post
title:  "Filtering - handling optional values using QueryDSL with Java 8"
date:   2016-05-14 19:45:31 +0530
categories: ["programming", "java"]
comments: true
---
## QueryDSL
[QueryDSL](http://www.querydsl.com/) is a nice library for producing typesafe, dynamic queries. It is a sane alternative to standard, hideous JPA Criteria API.

Spring Data provides a great way to create simple queries, but for more dynamic ones you have to go back to Criteria API or QueryDSL. Both are supported in Spring Data.

## The problem

The most common use case of dynamic queries is filtering. So let's say you have a list of blog Posts which have a bunch of properties and you want to filter it by all of them.

## Project setup

The project is based on Spring Boot, Spring Data with H2 Database, QueryDSL, Lombok as mandatory lib and Spock with DbUnit for testing. You can find [working example on my github](https://github.com/jakubdyszkiewicz/querydsl-optional-filter). 

First, let's create our domain objects.

{% highlight java %}
@Entity
@Data
public class Post {
    @Id
    private Integer id;
    private String author;
    private String title;
    private String body;
    private LocalDate date;

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(name = "post_tags", joinColumns = @JoinColumn(name = "post_id", referencedColumnName = "id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id", referencedColumnName = "id"))
    private List<Tag> tags;
}

@Entity
@Data
public class Tag {
    @Id
    private Integer id;
    private String name;
    @ManyToMany(mappedBy = "tags")
    private List<Post> posts;
}
{% endhighlight %}

### QueryDSL with Spring Data
Once everything is set up you can create repository extending `QueryDslPredicateExecutor`

{% highlight java %}
public interface PostRepository extends CrudRepository<Post, Integer>, QueryDslPredicateExecutor<Post> {
}
{% endhighlight %}

It adds methods like `Iterable<T> findAll(Predicate predicate);` or `Page<T> findAll(Predicate predicate, Pageable pageable);` to fetch entities by QueryDSL Predicate.

## Filtering

### PostFilter

We want to filter out list by author, title, date range (not just exact day of post) and list of tags. Tags are represented by list of Strings as it would be prefered way to query from frontend.

{% highlight java %}
@Data
public class PostFilter {
    private String author;
    private String title;
    private LocalDate from;
    private LocalDate to;
    private List<String> tags;
}
{% endhighlight %}

### Creating FilterBuilder - Take 1
Now we need a class to provide Predicate for given PostFilter that can later be used for `findAll(Predicate predicate)` method

{% highlight java %}
public class PostFilterOldBuilder implements PostFilterBuilder {

    private final QPost POST = QPost.post;

    public Predicate build(PostFilter filter) {
        BooleanBuilder builder = new BooleanBuilder(POST.isNotNull());
        if (!StringUtils.isEmpty(filter.getAuthor())) {
            builder.and(POST.author.containsIgnoreCase(filter.getAuthor()));
        }
        if (!StringUtils.isEmpty(filter.getTitle())) {
            builder.and(POST.title.containsIgnoreCase(filter.getTitle()));
        }
        if (filter.getFrom() != null) {
            builder.and(POST.date.after(filter.getFrom()));
        }
        if (filter.getTo() != null) {
            builder.and(POST.date.before(filter.getTo()));
        }
        if (!CollectionUtils.isEmpty(filter.getTags())) {
            builder.and(POST.tags.any().name.in(filter.getTags()));
        }
        return builder;
    }
}
{% endhighlight %}

Let's face it. That looks like a lot of boilerplate... and it is a small example! Imagine you have 20 fields (which is not that uncommon).

### Creating FilterBuilder - Take 2
We can leverage Java 8 method references and create a small fluent API.

{% highlight java %}
public class OptionalBooleanBuilder {

    private BooleanExpression predicate;

    public OptionalBooleanBuilder(BooleanExpression predicate) {
        this.predicate = predicate;
    }

    public <T> OptionalBooleanBuilder notNullAnd(Function<T, BooleanExpression> expressionFunction, T value) {
        if (value != null) {
            return new OptionalBooleanBuilder(predicate.and(expressionFunction.apply(value)));
        }
        return this;
    }

    public OptionalBooleanBuilder notEmptyAnd(Function<String, BooleanExpression> expressionFunction, String value) {
        if (!StringUtils.isEmpty(value)) {
            return new OptionalBooleanBuilder(predicate.and(expressionFunction.apply(value)));
        }
        return this;
    }

    public <T> OptionalBooleanBuilder notEmptyAnd(Function<Collection<T>, BooleanExpression> expressionFunction, Collection<T> collection) {
        if (!CollectionUtils.isEmpty(collection)) {
            return new OptionalBooleanBuilder(predicate.and(expressionFunction.apply(collection)));
        }
        return this;
    }

    public BooleanExpression build() {
        return predicate;
    }
}
{% endhighlight %}

and then use it like that

{% highlight java %}
public class PostFilterFluentBuilder implements PostFilterBuilder {

    private final QPost POST = QPost.post;

    public Predicate build(PostFilter filter) {
        return new OptionalBooleanBuilder(POST.isNotNull())
                .notEmptyAnd(POST.author::containsIgnoreCase, filter.getAuthor())
                .notEmptyAnd(POST.title::containsIgnoreCase, filter.getTitle())
                .notNullAnd(POST.date::after, filter.getFrom())
                .notNullAnd(POST.date::before, filter.getTo())
                .notEmptyAnd(POST.tags.any().name::in, filter.getTags())
                .build();
    }
}
{% endhighlight %}

Not only it is more readable but alao it is flexible as `OptionalBooleanBuilder` accepts QueryDSL `BooleanExpression` which most filtering expressions used for filtering inherit from.

### QueryDSL Web Support
If you are sending filter parameters from web (which you probably are) you can also use [Spring QueryDSL Web Support](http://docs.spring.io/spring-data/commons/docs/current/reference/html/#core.web.type-safe). It looks neat for basic usage, but customization looks a bit cumbersome to me.

## Testing
Does it even work? To check that, I created sample database state using DbUnit

{% highlight xml %}
<dataset>
    <tag id="1" name="tech" />
    <tag id="2" name="lifestyle" />
    <tag id="3" name="cooking" />

    <post id="1" author="Craig Larman" title="iPhone 7 leaked" date="2016-05-10"/>
    <post id="2" author="Stuart Halloway" title="10 things to buy before Christmas" date="2016-01-13"/>
    <post id="3" author="Louis Fleenor" title="Top 5 phones of 2015" date="2015-03-10"/>
    <post id="4" author="Billy Burrill" title="The perfect pizza" date="2016-05-15"/>
    <post id="5" author="Deidra Mott" title="Is Google Glass dead?" date="2016-09-10"/>
    <post id="6" author="Buffy Hellwig" title="Fried Chicken Grilled Cheese" date="2014-05-10"/>

    <post_tags post_id="1" tag_id="1" />
    <post_tags post_id="2" tag_id="1" />
    <post_tags post_id="2" tag_id="2" />
    <post_tags post_id="3" tag_id="1" />
    <post_tags post_id="3" tag_id="2" />
    <post_tags post_id="4" tag_id="3" />
    <post_tags post_id="5" tag_id="1" />
    <post_tags post_id="6" tag_id="3" />
</dataset>
{% endhighlight %}

and created Spring Integration Test using Spock and its wonderful support for Data Driven Testing

{% highlight groovy %}
@DatabaseSetup("/posts.xml")
@Unroll("should retrieve posts #ids given filter #postFilter")
def "should retrieve posts given filter"(Map postFilter, List ids) {
    expect:
        def predicate = filterBuilder.build(new PostFilter(postFilter))
        def posts = postRepository.findAll(predicate)
        posts*.id == ids
    where:
        postFilter                          | ids
        [:]                                 | [1, 2, 3, 4, 5, 6]
        [author: 'larman']                  | [1]
        [title: 'phone']                    | [1, 3]
        [from: LocalDate.of(2016, 1, 1)]    | [1, 2, 4, 5]
        [to: LocalDate.of(2016, 1, 1)]      | [3, 6]
        [tags: ['tech']]                    | [1,2,3,5]
        [tags: ['tech', 'cooking']]         | [1, 2, 3, 4, 5, 6]
        [author: 'larman', title: 'phone']  | [1]
}
{% endhighlight %}

It can be run directly from IDE or by `./gradlew test`

# Conclusion
Java 8 once again simplified code by using very basic functional approach. This little trick can be used not only for constructing QueryDSL Predicates but also for minimizing "wall" of repeatable null checking and action on value.