---
layout: post
title: "Filter"
---

To enable filtering with multiple variables, I have added a Specification to my project
# Specification
```
package Project.Java.Anime;

import jakarta.persistence.criteria.Predicate;
import org.springframework.data.jpa.domain.Specification;

import java.util.ArrayList;
import java.util.List;
public class AnimeSpecification {

    public static Specification<Anime> filter(String title, String miniText, Integer year) {
        return (root, query, criteriaBuilder) -> {
            List<Predicate> predicates = new ArrayList<>();

            if (title != null) {
                predicates.add(criteriaBuilder.equal(root.get("title"), title));
            }
            if (year != null){
                predicates.add(criteriaBuilder.equal(root.get("yearOfRelease"), year));
            }
            if (miniText != null) {
                predicates.add(criteriaBuilder.like(root.get("title"), "%" + miniText + "%"));
            }
            return criteriaBuilder.and(predicates.toArray(new Predicate[0]));
        };
    }
}

```
and initially built filter methods that used to work with a different structure that didn't allow using multiple filters simultaneously.
# Filter before Specification
Controller: 
```
    @GetMapping("/anime/filter")
    List<Anime> filter(@RequestParam String input, @RequestParam AnimeFilter filter) {
        switch (filter) {
            case BY_TITLE:
                return service.filterByTitle(input);
            case BY_GENRE:
                return service.filterByGenre(input);
            case BY_YEAR:
                return service.filterByYear(Integer.parseInt(input));
            case BY_CREATOR:
                return service.filterByCreator(input);
            case BY_SHORT_TEXT:
                return service.filterByText(input);
            case BY_STREAM:
                return service.filterBYStream(input);
            default:
                throw new  IllegalArgumentException("Ungültige Eingabe: " + input);
        }
    }
```
Service:
```
   public List<Anime> filterByText(String input) {
        return repository.findAll().stream()
                .filter(e -> e.getTitle().contains(input))
                .collect(Collectors.toList());
    }


    public List<Anime> filterByTitle(String input) {
        return repository.findAll().stream()
                .filter(anime -> anime.getTitle().equals(input))
                .toList();
    }

    public List<Anime> filterByYear(int input) {
        return repository.findAll().stream()
                .filter(anime -> anime.getYearOfRelease() == input)
                .toList();
    }
```
# Filter after Specification
Controller:
```
    @GetMapping("/anime")
    List<Anime> allOrWithFilter(@RequestParam(required = false) String title, @RequestParam(required = false) String miniText, @RequestParam(required = false) Integer year) {
       return service.filtered(title,miniText,year);
    }
```
Service: 
```
    public List<Anime> filtered(String title, String miniText, Integer year) {
        return repository.findAll(AnimeSpecification.filter(title, miniText, year));
    }
```
Now, with the Specification, I was able to combine the 'get all' method with the filtering options. 
This was made possible with the help of a colleague. 

Thanks for reading my blog!