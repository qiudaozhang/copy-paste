```java
List<List<Integer>> data = List.of(
        List.of(1, 2, 3),
        List.of(11, 22, 33)
);

List<Integer> list = data.stream().flatMap(List::stream).toList();
Integer reduce = list.parallelStream().reduce(0, Integer::sum);
```



