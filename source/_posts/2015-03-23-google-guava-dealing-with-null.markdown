---
layout: post
title: "Google Guava: 1. Dealing with Null"
date: 2015-03-23 14:00:40 +0800
comments: true
categories: [Java]
tags: [Java, Guava]  
keywords: Google Guava, Null, Optional, Enums, Objects  
description: Guava提供了处理null值的工具类，例如Optional，Enums，Objects。

---

> Please ref to the demo code: `git clone https://github.com/AlphaWang/guava-demo.git`


Many of Guava's utilities are designed to fail fast in the presence of null rather than allow nulls to be used. 
> com.alphawang.guava.ch1.optional.Test1_FailFast

```java
  // Lists.newArrayList()
  public static <E> ArrayList<E> newArrayList(E... elements) {
    checkNotNull(elements); 
    int capacity = computeArrayListCapacity(elements.length);
    ArrayList<E> list = new ArrayList<E>(capacity);
    Collections.addAll(list, elements);
    return list;
  }

```

Additionally, Guava provides a number of facilities both to make using `null` easier, when you must, and to help you avoid using `null`.

<!--more-->

## Null object pattern
see http://www.tutorialspoint.com/design_pattern/null_object_pattern.htm

- Consider if there is a natural "null object" that can be used. 

If it's an enum, add a constant to mean whatever you're expecting null to mean here. 

example 1, `java.math.RoundingMode` has an `UNNECESSARY` value to indicate "do no rounding, and throw an exception if rounding would be necessary."

example 2: use Unit.NONE instead of `null`.
```
public enum Unit {
   NONE("없음", "1"),

   // 길이(cm)
   MM("mm", "0.1"),
   CM("cm", "1"),
   M("m", "100"),
   KM("km", "100000"), ...
}
```

- Never return null. 

For example, return a empty list instead of null.
```
//BEFORE
public List<String> getNames(Long id) {
	if (id != null) {
	    return service.getNames(id);
	}
	return null;
}

//AFTER
public List<String> getNames(Long id) {
	if (id != null) {
	    return service.getNames(id);
	}
	return Collections.EMPTY_LIST;
}

```


## Optional

> com.alphawang.guava.ch1.optional.Test2_Optional

`Optional<T>` is a way of replacing a nullable T reference with a non-null value. 

Besides the increase in readability that comes from giving `null` a *name*, the biggest advantage of Optional is its idiot-proof-ness. **It forces you to actively think about the absent case if you want your program to compile at all, since you have to actively unwrap the Optional and address that case**. 

there are two main use cases of `Optional`.

- remind your client that the returned value can be null, can force he to check null case:
api:

```
public Optional<Object> api() {
    ...
    return Optional.fromNullable(nullableObj); // return Optional instead of nullableObj
}
```
client:
```
Optional<Object> optional = api();
if (optional.isPresent()) { // force null-check
} 
```

- return a default value if the object is null:

```
Object obj = Optional.fromNullable(nullableObj).or(defaultObj);
```

## Enums

`Enums#getIfPresent` can help to get an Optional<Enum> from a enum value.

```
        // BEFORE
        VitaminLocale vitaminLocale = VitaminLocale.valueOf("ko_KR");
        if (vitaminLocale != null) {

        }

        // AFTER
        Optional<VitaminLocale> locale = Enums.getIfPresent(VitaminLocale.class, "ko_KR");
        if (locale.isPresent()) {
            locale.get();
        }
```

## Objects

> com.alphawang.guava.ch1.optional.Test3_Objects

`MoreObjects.firstNonNull` is an alternative of `Optional.or`, both can return a default value if the object is null.

```
MoreObjects.firstNonNull(cityService.getNullableCity(1L), defaultCity);
```


## Preconditions

> com.alphawang.guava.ch1.optional.Test4_Preconditions

Of course we can fail fast in the presence of null, just like guava code. Guava provides `Preconditions.checkArgument` and `Preconditions.checkNotNull` to do this.

```
    public City save(City city) {
        Preconditions.checkArgument(city != null, "IllegalArgumentException: city is null");
        // or
        city = Preconditions.checkNotNull(city, "NullPointerException: city is null");

        return cityRepository.save(city);
    }
```


## Reference
- Guava User Guide : https://code.google.com/p/guava-libraries/wiki/GuavaExplained
- Getting Started with Google Guava : http://pan.baidu.com/s/1o6LZJf0 



