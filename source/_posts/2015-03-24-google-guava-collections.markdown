---
layout: post
title: "Google Guava: 2. Collections"
date: 2015-03-24 14:00:40 +0800
comments: true
categories: [Java]
tags: [Java, Guava]  
keywords: Google Guava, Collections, Lists, Maps, FluentIterables  
description: Guava提供了处理null值的工具类。
---

> Please ref to the demo code: `git clone https://github.com/AlphaWang/guava-demo.git`

The Guava library has its history rooted in working with collections, starting out as `google-collections`. The Google Collections Library has long since been abandoned, and all the functionality from the original library has been merged into Guava. 

<!--more-->

## Create
> com.alphawang.guava.ch2.collection.Test1_Create

### Create List/Set

```
        // Before
        List<String> list1 = new ArrayList<String>(2);
        list1.add("A");
        list1.add("B");

        // After
        List<String> list2 = Lists.newArrayList(); //Lists.newArrayListWithCapacity(2)
        list2.add("A");
        list2.add("B");
```

- Lists.newArrayList()
- Lists.newArrayList(E...)
- Lists.newArrayList(java.lang.Iterable<? extends E>)
- Lists.newArrayListWithCapacity
- Lists.newArrayListWithExpectedSize()
- Lists.newLinkedList()
- Sets.newHashSet()
- Sets.newTreeSet()


### Create Map

```
        // Before
        Map<String, String> map1 = new HashMap<String, String>();

        // After
        Map<String, String> map2 = Maps.newHashMapWithExpectedSize(2);  // Maps.newHashMap();
```

- Maps.newHashMap()
- Maps.newHashMapWithExpectedSize()
- Maps.newLinkedHashMap()
- Maps.newTreeMap()


### Immutable Collections

```
        // Before
        List<String> list = new ArrayList<String>();
        list.add("A");
        list.add("B");
        List<String> list1 = Collections.unmodifiableList(list);

        // After
        List<String> list2 = ImmutableList.of("A", "B");
```

```
        // Before
        Map<String, String> map = new HashMap<String, String>();
        map.put("A", "1");
        map.put("B", "2");
        Map<String, String> map1 = Collections.unmodifiableMap(map);

        // After
        Map<String, String> map2 = ImmutableMap.of("A", "1", "B", "2");
```

- ImmutableList.of()
- ImmutableList.copyOf(java.lang.Iterable<? extends E>)
- ImmutableMap.of()
- ImmutableMap.copyOf(Map<? extends K, ? extends V> map)


Immutable objects have many advantages, including:

1. Safe for use by untrusted libraries.
2. Thread-safe: can be used by many threads with no risk of race conditions.
3. Doesn't need to support mutation, and can make time and space savings with that assumption. All immutable collection implementations are more memory-efficient than their mutable siblings. 
4. Can be used as a constant, with the expectation that it will remain fixed.



## Transform and Filter
> com.alphawang.guava.ch2.collection.Test2_Transform
> com.alphawang.guava.ch2.collection.Test3_Filter

Transform List:
```
    /**
     * Lists.transform
     */
    @Test
    public void transformList() {
        // before:
        List<String> cityNamesBefore = new ArrayList<>();
        for (City city : cities) {
            System.out.println("-- normal transform: " + city);
            cityNamesBefore.add(city.getName());
        }

        // or Iterables.transform()
        List<String> cityNames = Lists.transform(cities, new Function<City, String>() {
            @Override
            public String apply(City input) {
                System.out.println("-- guava transform: " + input);
                return input.getName();
            }
        });

        Assert.assertTrue(cityNames.size() == 4);
        System.out.println(cityNames);   // lazy
    }
```

```
    /**
     * FluentIterable.transform
     */
    @Test
    public void transformFluent() {
        List<Integer> cityNameLength = FluentIterable.from(cities)
            .transform(cityNameFunction)
            .transform(stringLengthFunction)
            .toList();

        System.out.println(cityNameLength);
        Assert.assertTrue(cityNameLength.size() == 4);
    }

    /**
     * Functions.compose
     */
    @Test
    public void functions() {
        List<Integer> cityNameLength = Lists.transform(cities, Functions.compose(stringLengthFunction, cityNameFunction));

        System.out.println(cityNameLength);
        Assert.assertTrue(cityNameLength.size() == 4);
    }
```

Transform Map:

```
    @Test
    public void transformMapEntry() {
        Map<City, String> map = Maps.transformEntries(cityLocaleMap, new Maps.EntryTransformer<City, VitaminLocale, String>() {
            @Override
            public String transformEntry(City key, VitaminLocale value) {
                return key.getPopulation() + "|" + value.getKoreaTitle();
            }
        });

        System.out.print(map);
        Assert.assertTrue(map.size() == cityLocaleMap.size());
    }

    @Test
    public void transformMapValue() {
        Map<City, String> map = Maps.transformValues(cityLocaleMap, new Function<VitaminLocale, String>() {
            @Override
            public String apply(VitaminLocale input) {
                return input.getEnglishTitle();
            }
        });

        System.out.print(map);
        Assert.assertTrue(map.size() == cityLocaleMap.size());
    }

```

Filter:

```
    /**
     *  Iterables.filter()
     */
    @Test
    public void iterablesFilter() {
        Iterable<City> largeCities = Iterables.filter(cities, new Predicate<City>() {
            @Override public boolean apply(City input) {
                return input.getPopulation() > 1000L;
            }
        });

        System.out.println(largeCities);
        Assert.assertTrue(Iterables.size(largeCities) == 2);
    }

    /**
     * FluentIterable.filter()
     */
    @Test
    public void fluentIterableFilter() {

        List<City> result = FluentIterable.from(cities)
            .filter(populationPredicate)
            .filter(namePredicate)
            .toList();

        Assert.assertTrue(result != null);
    }

    /**
     * Predicates.and()
     */
    @Test
    public void predicates() {
        Iterable<City> result = Iterables.filter(cities, Predicates.and(populationPredicate, namePredicate));

        Assert.assertTrue(!Iterables.isEmpty(result));
    }




    @Test
    public void filterMap() {
        Map<String, City> nameCityMap = Maps.uniqueIndex(cities, new Function<City, String>() {
            @Override
            public String apply(City input) {
                return input.getName();
            }
        });
        System.out.println(nameCityMap);


        Map<String, City> filteredMap = Maps.filterEntries(nameCityMap, new Predicate<Map.Entry<String, City>>() {
            @Override
            public boolean apply(Map.Entry<String, City> input) {
                return input.getKey().startsWith("S")
                    && input.getValue().getPopulation() > 1000L;
            }
        });

        Assert.assertTrue(filteredMap.size() == 1);
    }
```


## Convert between List and Map

> com.alphawang.guava.ch2.collection.Test4_Convert

### List to Map

1. `Maps.uniqueIndex` method uses Function to generate keys from the given values. 
2. `Maps.asMap` method takes a set of objects to be used as keys, and Function is applied to each key object to generate the value for entry into a map instance.
3. `Maps.toMap` returns ImmutableMap.

 
```
    Function<City, Long> cityToIdFunction = new Function<City, Long>() {
            @Override
            public Long apply(City input) {
                return input.getId();
            }
        };

    @Test
    public void listToMapAsValue() {
        Map<Long, City> idCityMap = Maps.uniqueIndex(cities, cityToIdFunction);
    }

    @Test
    public void listToMapAsKey() {
        Map<City, Long> cityIdMap = Maps.toMap(cities, cityToIdFunction);
        // or
        cityIdMap = FluentIterable.from(cities).toMap(cityToIdFunction);
    }
```


### Map to List

```
    @Test
    public void mapToList() {
        Map<City, String> cityCommentMap = ImmutableMap.of(
            new City(1L, "Shanghai", 1360L), "big",
            new City(2L, "Beijing", 1020L), "dirty"
        );

        List<String> cityComments = FluentIterable.from(cityCommentMap.entrySet())
            .transform(new Function<Map.Entry<City,String>, String>() {
                @Override
                public String apply(Map.Entry<City, String> input) {
                    return input.getKey().getName() + " IS " + input.getValue();
                }
            }).toList();
        System.out.println("\nmapToList:\n" + cityComments);
    }
```


## Utils

```
Iterables.getOnlyElement() 
Iterables.concat()

Sets.unin()
Sets.difference()
Sets.intersection()
Lists.partition()
```


## Reference
- Guava User Guide : https://code.google.com/p/guava-libraries/wiki/GuavaExplained
- Getting Started with Google Guava : http://pan.baidu.com/s/1o6LZJf0 





