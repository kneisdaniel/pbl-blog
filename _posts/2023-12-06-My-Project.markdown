---
layout: post
title: "Code, Navigate, Discover: My Experience Crafting an Anime Lexicon in Spring Boot"
---
In this blog, I will introduce you to my project and delve into some problems I encountered during it's development. This project is part of a 10-month Java programming course offered by [everyone codes](https://everyonecodes.io/), and we had 6 weeks to work on it.

## Filter

To allow to filter the list of Anime's I created this code:

Service:
```
   public List<Anime> filterByText(String input) {
        return repository.findAll().stream()
                .filter(e -> e.getTitle().contains(input))
                .collect(Collectors.toList());
    }
```
This code schould allow the user to filter the list so that only the animes that contain the user's input are included.
```
    public List<Anime> filterByYear(int input) {
        return repository.findAll().stream()
                .filter(anime -> anime.getYearOfRelease() == input)
                .toList();
    }
```
This code is intended to allow the user to filter the list so that only the anime in it are of the same year as the input.

This is the Controller for the filter methodes:
```
 @GetMapping("/anime/filter")
    List<Anime> filter(@RequestParam String input, @RequestParam AnimeFilter filter) {
        switch (filter) {
            case BY_YEAR:
                return service.filterByYear(Integer.parseInt(input));
            case BY_SHORT_TEXT:
                return service.filterByText(input);
            default:
                throw new  IllegalArgumentException("Ungültige Eingabe: " + input);
        }
    }
```
The Controlller works with enums to find the right filter for the user. For the manual testing I used Postman. The way I used for filtering has worked, but this is not right it don't allow the user to filter with more Parameters.

With this realization I searched other way to filter with more parameters and that the list filtered in the database and not in the service. 

After two days I found a new way to let the user use more filter options and started to write the code.

# Specification

Now with the specification I can fix the two problems. First the user can filter with more options in the same time. Second the filtering is carried out in the database and no longer in the service.

To use the specification we need this code in the reposetory: 
```
@Repository
public interface AnimeRepository extends JpaRepository<Anime, Long>, JpaSpecificationExecutor<Anime> {
}
```
Only with the `JpaSpecificationExecutor` can the code work, by starting without the `JpaSpecificationExecutor` throws a Exception.

Now we look in the AnimeSpecification: 

```
 public static Specification<Anime> filter(String miniText, Integer year) {
        return (root, query, criteriaBuilder) -> {
            List<Predicate> predicates = new ArrayList<>();

            if (year != null){
                predicates.add(criteriaBuilder.equal(root.get("yearOfRelease"), year));
            }
            if (miniText != null) {
                predicates.add(criteriaBuilder.like(root.get("title"), "%" + miniText + "%"));
            }
            return criteriaBuilder.and(predicates.toArray(new Predicate[0]));
        };
    }
```
This specification class has two options to filter the animes per included text and per the same year.

Contoller with Specivication: 
```
    @GetMapping("/anime")
    List<Anime> allOrWithFilter(@RequestParam(required = false) String miniText, @RequestParam(required = false) Integer year) {
       return service.filtered(miniText,year);
    }
```
This is how the Controller looks like with the Specification.

Service with Spazifikation:
```
    public List<Anime> filtered(String title, String miniText, Integer year) {
        return repository.findAll(AnimeSpecification.filter(title, miniText, year));
    }
```
And this is the Service with the Specification.

Thanks for reading my Blog.