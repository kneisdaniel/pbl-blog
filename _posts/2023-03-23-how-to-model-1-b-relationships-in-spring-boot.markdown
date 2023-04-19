---
layout: post
title: "Modelling 1:N relationships in Spring Boot"
---
Hi! My name is Max Mustermann and I recently finished a 10-month Java programming course at [everyone codes](https://everyonecodes.io/). In order to practice our newly-gained skills in Java and Spring Boot, we had 6 weeks to work on a small project of our choice. As an avid language learner, I decided to write a small clone of Anki, a program that allows you to create and review flashcards. 

In this blog post, I want to talk about the difference between the `@OneToMany` and `@ManyToOne` annotation. It took me a lot of research to properly understand the difference and when to use which one, so I hope that I can save you some hours of tedious research by summarizing my findings in this blog post! 

## Requirements
So, to provide a bit of background, the requirements for my app were as follows:
- The user should be able to add and delete decks.
- The user should be able to add, update and delete cards in decks. Each card can only be part of one deck.
- Each card should contain the word in the native language, the word in the target language and optionally an example sentence in the target language.
- The user should be able to review cards, i.e. there needs to be some mechanism to keep track of how well a person learned a card.

## The problem

At some point while trying to add the necessary entities to my program, the two entities `Deck` and `Card` looked as follows:

```java
@Entity
public class Deck {
    @Id
    @GeneratedValue
    private Long id;
    @NotEmpty
    private String name;
}
```

```java
@Entity
public class Card {
    @Id
    @GeneratedValue
    private Long id;
    @NotEmpty
    private String sourceWord;
    @NotEmpty
    private String targetWord;
    private String targetSentence;
}
```

Simple enough, right? We have two very basic entities with a couple of fields. But now the questions arises: What is the best way to define the relationship between the two? As mentioned in the requirements, a deck should consist of (many) cards, and a card should belong to exactly one deck (the latter requirement might not be necessary in theory for a flashcard program, but in order to not make things too complicated, I decided to impose this constraint). So, how do we represent this relationship in Spring?

## The three types of relationships

In database theory, we distinguish between **three different types of relationship** between entities. Let's say that our first entity is called `X` and our second one `Y`. The possible relationships are:
- **1:1 relationship** (one-to-one)
  - **Explanation**: A `X` has exactly one `Y`, and a `Y` has exactly one `X`.
  - **Example**: A `Person` has exactly one `SocialSecurityNumber`, and a `SocialSecurityNumber` belongs to exactly one `Person`.
- **1:N relationship** (one-to-many / many-to-one)
  - **Explanation**: A `X` can have many `Y`, but a `Y` only has exactly one `X`.
  - **Example**: A `Customer` can have many `Order`s, but an `Order` belongs to exactly one `Customer`.
- **N:M relationship** (many-to-one)
  - **Exaplanation**: A `X` can have many `Y`, and a `Y` can have many `X`.
  - **Example**: A `Student` can participate in many `Course`s, and a `Course` can be attended by many `Student`s.

So, given that information, can you already guess which type of relationship is appropriate in our case?

Exactly, the `1:N` relationship! A card belongs to exactly one deck, but a deck can have multiple cards. But how do we model this in Spring Boot? Well, we have two options.

## `OneToMany` vs `ManyToOne`
So in order to model this relationship, we either have to add a `List<Card>` field on the side of the `Deck` and add a `@OneToMany` annotation, or we add a field `Deck` on the side of the `Card` and a `@ManyToOne`.

But what is the difference between them? Let's take a closer look at what each of these annotations does to get a better understanding of what's happening under the hood.

### Option 1
Let's consider the first option, namely adding the reference on the side of the deck. This is what it would look like:
```java
@Entity
public class Deck {
    @Id
    @GeneratedValue
    private Long id;
    @NotEmpty
    private String name;
    @OneToMany
    private List<Card> cards;
}
```

To understand what's really happening in the background, we can let Spring show us the SQL statements that are actually being executed in the background by adding the following lines into our `application.properties`:

```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

The output is as follows (I removed the part where the constraints on the foreign keys are added):

```
Hibernate: 
    
    create table card (
       id bigint not null,
        source_word varchar(255),
        target_sentence varchar(255),
        target_word varchar(255),
        primary key (id)
    )
Hibernate: 
    
    create table deck (
       id bigint not null,
        name varchar(255),
        primary key (id)
    )
Hibernate: 
    
    create table deck_cards (
       deck_id bigint not null,
        cards_id bigint not null
    )

-- snip --

```

So basically, the `deck` and `card` tables are created as if there were no connection between them at all! Instead, a new table called `decks_cards` is created, which contains the IDs of all of the relations between the two. So for example, let's say we have two decks with the IDs `1` and `2`. Deck `1` contains the cards with the IDs `1`, `2` and `3`, and Deck `2` contains the card with the ID `4`. Then, the table would look as follows:

| deck_id | card_id |
| ------  | ------- |
| 1       | 1       |
| 1       | 2       |
| 1       | 3       |
| 2       | 4       |

### Option 2
So what happens if we define the relationship on the other side instead? Well, let's try it:

```java
@Entity
public class Card {
    @Id
    @GeneratedValue
    private Long id;
    @NotEmpty
    private String sourceWord;
    @NotEmpty
    private String targetWord;
    private String targetSentence;
    @ManyToOne
    private Deck deck;
}
```

The output is now:
```
Hibernate: 
    
    create table card (
       id bigint not null,
        source_word varchar(255),
        target_sentence varchar(255),
        target_word varchar(255),
        deck_id bigint,
        primary key (id)
    )
Hibernate: 
    
    create table deck (
       id bigint not null,
        name varchar(255),
        primary key (id)
    )
    
-- snip --
```

Notice something different? The third table is gone! And instead, the corresponding deck ID is stored as a property in the `card` table.

### Choosing between the two
So, which option should you choose? Well, this entirely depends on your use case and how you want to be able to "navigate" your entities. If, given a deck, you need a convenient way of accessing all of the corresponding cards, you should put the reference on the side of the deck. If instead, it is more important to access the corresponding deck from a card, you should put the reference on the side of the card. In the case of my project, the first one was more important, so that's the one I chose.

By the way, in many cases, both of the above-mentioned directions might be important. In this case, you can also define a **bidrectional relationship**. For our example, it would look something like this:

```java
@Entity
public class Deck {
    @Id
    @GeneratedValue
    private Long id;
    @NotEmpty
    private String name;
    @OneToMany(mappedBy = "deck")
    private List<Card> cards;
}
```

```java
@Entity
public class Card {
    @Id
    @GeneratedValue
    private Long id;
    @NotEmpty
    private String sourceWord;
    @NotEmpty
    private String targetWord;
    private String targetSentence;
    @ManyToOne
    private Deck deck;
}
```

Note the `mappedBy` attribute in the `@OneToMany` relationship. We need this so that Spring knows what the corresponding field of the `Deck` on the `Card`'s side is.

Thanks for reading my blog, and I hope it helped you!