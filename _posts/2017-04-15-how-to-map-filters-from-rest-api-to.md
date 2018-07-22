---
layout: post
title: How to map filters from a REST API to a Spring Data JPA
date: 2017-04-15
tags: [ Spring Boot, Spring Data, Spring JPA, Hibernate ]
---
In this guide we are going to map filters set via a REST API to Spring Data JPA repository. The filters via a REST API look like:

```
../rest/items?filter=field1>10&field2=something
```

Using this tutorial, you will be able to extend this framework so you can use any character to map it to any filter operator. In the above example, we are using only '>' and '='. See our Item controller:

```java
@RestController
@RequestMapping("/items")
public class ItemController {

	private final ItemService itemService;

	@Autowired
	public ItemController(ItemService itemService) {
		this.itemService = itemService;
	}

	@GetMapping
	public List<Item> listItems(@RequestParam(value = "filter", defaultValue = "") List<String> filters) {
		return itemService.getItems(filters);
	}	
}
```

Ideally, we should wrap the filters, sort settings and anything else within a single request. At this tutorial, for simplicity, we will only have a list of string. See our Item service getItems method:

```java
@Service
@Transactional
public class ItemServiceImpl implements ItemService {

	private final ItemRepository itemRepository;

	@Autowired
	public ItemServiceImpl(ItemRepository itemRepository) {
		this.itemRepository = itemRepository;
	}

	@Override
	public List<Item> getItems(List<String> filters) {
		// @formatter:off
		return FindAllBuilder.usingRepository(itemRepository)
					.filterBy(request.getFilters())
					.findAll(Page.FIRST, Limit.MAX_SIZE);
		// @formatter:on
	}
}
```

We are using a builder to make a enricher "findAll" call to any repository. See the implementation:

```java
public class FindAllBuilder<T extends PagingAndSortingRepository<T,?> & JpaSpecificationExecutor<T> {

	private final R repository;

	private Specifications<T> filters;

	private Sort sort = Sort.unsorted();

	public static <T extends PagingAndSortingRepository<T,?> & JpaSpecificationExecutor<T>> FindAllBuilder<T,R> 
usingRepository(R repository) {
		return new FindAllBuilder<>(repository);
	}

	private FindAllBuilder(R repository) {
		this.repository = repository;
	}

	public List<T> findAll(int page, int limit) {
		return new LinkedList<T>(repository.findAll(filters, PageRequest.of(page, limit, sort)).getContent());
	}

	public FindAllBuilder<T,R> filterBy(List<String> listFilters) {
		Optional<Specification<T>> opFilters = EntitySpecificationBuilder.parse(listFilters);
		if (opFilters.isPresent()) {
			if (filters == null) {
				filters = Specifications.where(opFilters.get());
			} else {
				filters = filters.and(opFilters.get());
			}
		}

		return this;
	}

	public FindAllBuilder<T,R> sortBy(String orderBy, String orderDir) {
		if (StringUtils.isNotEmpty(orderBy)) {
			sort = Sort.by(Direction.fromOptionalString(orderDir).orElse(Direction.ASC), orderBy);
		}

		return this;
	}
}
```

Key notes:

* This builder supports sorting by a field (Out of scope for this tutorial).

* The repository has to implement the interfaces PagingAndSortingRepository (to enable paging and sorting) and JpaSpecificationExecutor (to enable specifications). Doing this, we will see many more overloaded findAll methods in the repository with more options.

* We delegate to the EntitySpecificationBuilder builder the mapping from a list of filters to Spring Data Specification. The Spring Data Specification is the way that Spring Data JPA enables you to make criteria over entities. For example, mapping each filter such as 'name=John' to a Spring Data specification with a criteria for the field "name" and it is equal to "John".

```java
public class EntitySpecificationBuilder<T> {

	public static <T> Optional<Specification> parse(List<String> filters) {
		if (filters == null || filters.isEmpty()) {
			return Optional.empty();
		}

		List<Specification> criterias = mapSpecifications(filters);
		if (criterias.isEmpty()) {
			return Optional.empty();
		}

		Specification<T> root = Specifications.where(criterias.get(0));
		for (int index = 1; index < criterias.size(); index++) {
			root = Specifications.where(root).and(criterias.get(index));
		}
		return Optional.of(root);
	}

	private static <T> List<Specification> mapSpecifications(List<String> filters) {
		return filters.stream().map(str -> {
			for (FilterOperation op : FilterOperation.values()) {
				int index = str.indexOf(op.getOperationName());
				if (index > 0) {
					String key = str.substring(0, index);
					String value = str.substring(index + op.getOperationName().length());
					return (Specification<T>) (root, query, cb) -> op.build(cb, root, key, value);
				}
			}

			return null;
		}).filter(s -> s != null).collect(Collectors.toList());
	}
}
```

This builder is very basic and is looping among the list of filters trying to look for a matching filter and then split the left and right sides of the filter. The magic happens in the enum FilterOperator:

```java
public enum FilterOperation {
	// @formatter:off
	LESS_EQUAL_THAN("<=", (b, k, v) -> b.lessThanOrEqualTo(k, v)),
	GREATER_EQUAL_THAN(">=", (b, k, v) -> b.greaterThanOrEqualTo(k, v)),
	CONTAINS(":", (b, k, v) -> b.like(k, b.literal("%" + v + "%"))),
	GREATER_THAN(">", (b, k, v) -> b.greaterThan(k, v)),
	LESS_THAN("<", (b, k, v) -> b.lessThan(k, v)),
	EQUALS("=", (b, k, v) -> b.equal(k, v));
	// @formatter:on

	private final String operationName;
	private final FilterPredicateFunction operation;

	FilterOperation(String operationName, FilterPredicateFunction operation) {
		this.operationName = operationName;
		this.operation = operation;
	}

	public String getOperationName() {
		return operationName;
	}

	public Predicate build(CriteriaBuilder builder, Root entity, String key, Object value) {
		return operation.predicate(builder, entity.get(key), value.toString());
	}

	static FilterOperation parse(String str) {
		for (FilterOperation filter : FilterOperation.values()) {
			if (StringUtils.equals(str, filter.getOperationName())) {
				return filter;
			}
		}

		throw new WrongFilterException(String.format("Filter operation '%s' not found", str));
	}

	@FunctionalInterface
	interface FilterPredicateFunction {
		Predicate predicate(CriteriaBuilder builder, Path<String> key, String value);
	}
}
```

See the supported filters and the matching Spring Data Specification. You can easily use your custom operators to match to a concrete Spring Data criteria or add yourselves just adding:

```java
MY_CUSTOM_OPERATOR("$$", (b, k, v) -> b.anyCriteria(k, v)),
```

Where "b" is a Spring Data CriteriaBuilder where we have lot of possible criteria and we can combine them as we like.

As a summary, we mapped a REST API filter to a Spring Data criteria with no coding in our controllers, services or repositories. We isolated this mapping in an enum that we can easily understand, maintain and extend. Our framework is easily extensible to support sorting and paging thanks to builders patterns.

If you need some working example or test examples, just ping me out.