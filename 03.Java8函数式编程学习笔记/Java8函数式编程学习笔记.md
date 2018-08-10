## Lambda

```java
1.Predicate<T> 参数T 返回类型Boolean
2.Consumer<T> 参数T 返回类型void
3.Funcation<T, R> 参数T 返回类型R
4.Supplier<T> 参数None 返回类型T
5.UnaryOperator<T> 参数T 返回类型T 
6.BinaryOperator<T> 参数T, T 返回类型T
```

- 默认方法

  ```reStructuredText
  1.类胜于接口，如果在继承链中有方法体或抽象的方法声明就可以忽略接口中定义的方法
  2.子类胜于父类，如果一个接口继承了另一个接口，且两个接口都定义了一个默认方法则子类胜出
  3.如果上面两条规则都不适用，那么子类需要实现该方法，或将该方法声明为抽象方法流式编程
  ```

- `Optional` 判空

  ```java
  public static void main(String[] args) {
      Optional.ofNullable("Roberto").ifPresent(System.out::println);
      System.out.println(Optional.ofNullable("Roberto").map(String::length).orElse(-1));
  }
  
  ofNullable
  isPresent 是否有值
  orElse 当Optional对象为空时，提供一个备选值 
  ```


## 流式编程

### 流创建

```java
1.Collection.stream();

2.Arrays.stream();

3.Stream stream = Stream.of();

4.IntStream/LongStream.rang()/rangeClosed()

5.Random.ints()/longs()/doubles()

6.Stream.generate()/iterate()
```

### 常用API    

#### **Terminal** 

- `forEach/forEachOrdered` 遍历

  ```java
  public static void main(String[] args) {
      // 使用并行流
      Stream.of("Roberto", "Henry", "DreamT").parallel().forEach(str -> System.out.println(str));
  
      // 使用forEachOrdered保证顺序
      Stream.of("Roberto", "Henry", "DreamT").parallel().forEachOrdered(str -> System.out.println(str));
  }
  ```

- `toArray/collect` 转为其他数据结构

  ```java
  public static void main(String[] args) {
      // 1.Array
      String[] strArray = Stream.of("Roberto", "Henry", "DreamT").toArray(String[]::new);
  
      // 2.Collection
      Set<String> collect = Stream.of("Roberto", "Henry", "DreamT").collect(Collectors.toSet());
      List<String> collect2 = Stream.of("Roberto", "Henry", "DreamT").collect(Collectors.toList());
      HashSet<String> collect3 = Stream.of("Roberto", "Henry", "DreamT").collect(Collectors.toCollection(HashSet::new));
      ArrayList<String> collect4 = Stream.of("Roberto", "Henry", "DreamT").collect(Collectors.toCollection(ArrayList::new));
  
      // 3.String
      String str = Stream.of("Roberto", "Henry", "DreamT").collect(Collectors.joining(", ", "[", "]")).toString();
  }
  ```

- `reduce()` 规约操作

  ```java
  public static void main(String[] args) {
      // 不带初始化值的Reduce
      Optional<String> reduce = Stream.of("Roberto", "Henry", "DreamT").reduce((s1, s2) -> s1 + "|" + s2);
      reduce.ifPresent(str -> System.out.println(str));
  
      // 带初始化值的Reduce
      String reduceStr = Stream.of("Roberto", "Henry", "DreamT").reduce("", (s1, s2) -> s1 + "|" + s2);
      System.out.println(reduceStr);
  }
  ```

- `max()/min()/count()` 最大值/最小值/计算总和

  ```java
  public static void main(String[] args) {
      List<String> strList = Arrays.asList("Roberto", "Henry", "DreamT");
      System.out.println(strList.stream().count());
      System.out.println(strList.stream().min(Comparator.comparing(String::length)).get());
      System.out.println(strList.stream().max(Comparator.comparing(String::length)).get());
  }
  ```

- 匹配查找

  ```reStructuredText
  AnyMatch()   流中一个元素能符合判断规则即为true
  AllMatch()   流中所有元素能符合判断规则即为true
  NoneMatch()  流中所有元素都不符合判断即为true
  findAny()    返回当前流中的任意元素 方法返回结果为Optional<T>
  findFirst()  返回当前流中的第一个元素 方法返回结果为Optional<T>
  ```

#### **Intermediate** 

- `map` 转换 转换前后`Stream`中元素的个数不会改变，但元素的类型取决于转换之后的类型 

  ```java
  public static void main(String[] args) {
      Stream<String> stream = Stream.of("Roberto", "Henry", "DreamT");
      stream.map(str -> str.toLowerCase()).forEach(str -> System.out.println(str));
  }
  ```

- `flatMap()` 转换 转换前后元素的个数和类型都可能会改变 适用于一对多的场景

  ```java
  public static void main(String[] args) {
      Stream<List<Integer>> stream = Stream.of(Arrays.asList(1,2), Arrays.asList(3, 4, 5));
      stream.flatMap(list -> list.stream()).forEach(i -> System.out.println(i));
  }
  ```

- `filter()` 过滤

  ```java
  public static void main(String[] args) {
      Stream<String> stream = Stream.of("Roberto", "Henry", "DreamT");
      stream.filter(str -> str.length() == 5).forEach(str -> System.out.println(str));
  }
  ```

- `peek` 遍历 常用于`Debug` 用法与`forEach`类似

  ```java
  public static void main(String[] args) {
      List<String> strList = Arrays.asList("Roberto", "Henry", "DreamT");
      strList.stream().peek(str -> System.out.println(str)).count();
  }
  ```

- `distinct()` 去重

  ```java
  public static void main(String[] args) {
      Stream<String> stream = Stream.of("Roberto", "Henry", "Henry");
      stream.distinct().forEach(str -> System.out.println(str));
  }
  ```

- `sorted()` 排序

  ```java
  public static void main(String[] args) {
      Stream<String> stream = Stream.of("Roberto", "Henry", "DreamT");
      stream.sorted(Comparator.comparingInt(String::length)).forEach(str -> System.out.println(str));
  }
  ```

- `limit()` 截断

  ```java
  public static void main(String[] args) {
      List<String> strList = Arrays.asList("Roberto", "Henry", "DreamT");
      strList.stream().limit(2).forEach(str -> System.out.println(str));
  }
  ```

- `skip()` 跳过

  ```java
  public static void main(String[] args) {
      List<String> strList = Arrays.asList("Roberto", "Henry", "DreamT");
      strList.stream().skip(2).forEach(str -> System.out.println(str));
  }
  ```

### 并行流

- `parallel()` 并行流

- `sequential()` 串行流

  ```java
  1.多次条用parallel/sequential，以最后一次调用为准
  2.并行流使用的线程池是ForkJoinPool.commonPool，默认线程数是当前机器的CPU个数
  3.并行不一定比串行快 要根据实际情况选择
  ```

- 使用自己的线程池不使用默认线程池，防止任务被阻塞

  ```java
  public static void main(String[] args) throws InterruptedException {
      ForkJoinPool forkJoinPool = new ForkJoinPool(20);
      forkJoinPool.submit(() -> IntStream.range(1, 100).parallel().peek(number -> System.out.println(number)).count());
      forkJoinPool.shutdown();
      synchronized (forkJoinPool) {
          forkJoinPool.wait();
      }
  }
  ```

### 收集器

```java
public class CollectorTest {
    public static List<Student> students;

    static {
        students = Arrays.asList(
                new Student("小明", 10, Gender.MALE, Grade.ONE),
                new Student("大明", 9, Gender.MALE, Grade.THREE),
                new Student("小白", 8, Gender.FEMALE, Grade.TWO),
                new Student("小黑", 13, Gender.FEMALE, Grade.FOUR),
                new Student("小红", 7, Gender.FEMALE, Grade.THREE),
                new Student("小黄", 13, Gender.MALE, Grade.ONE),
                new Student("小青", 13, Gender.FEMALE, Grade.THREE),
                new Student("小紫", 9, Gender.FEMALE, Grade.TWO),
                new Student("小王", 6, Gender.MALE, Grade.ONE),
                new Student("小李", 6, Gender.MALE, Grade.ONE),
                new Student("小马", 14, Gender.FEMALE, Grade.FOUR),
                new Student("小刘", 13, Gender.MALE, Grade.FOUR));
    }

    public static void main(String[] args) {
        // 得到所有学生的年龄列表
        Set<Integer> ages = students.stream().map(Student::getAge).collect(Collectors.toCollection(TreeSet::new));
        System.out.println("所有学生的年龄:" + ages);

        // 统计汇总信息
        IntSummaryStatistics agesSummaryStatistics = students.stream().collect(Collectors.summarizingInt(Student::getAge));
        System.out.println("年龄汇总信息:" + agesSummaryStatistics);

        // 分块统计信息
        Map<Boolean, List<Student>> genders = students.stream().collect(Collectors.partitioningBy(s -> s.getGender() == Gender.MALE));
        MapUtils.verbosePrint(System.out, "男女学生列表", genders);

        // 分组统计信息
        Map<Grade, List<Student>> grades = students.stream().collect(Collectors.groupingBy(Student::getGrade));
        MapUtils.verbosePrint(System.out, "学生班级列表", grades);

        // 得到所有班级学生的个数
        Map<Grade, Long> gradesCount = students.stream().collect(Collectors.groupingBy(Student::getGrade, Collectors.counting()));
        MapUtils.verbosePrint(System.out, "班级学生个数列表", gradesCount);
    }
}

@Getter
@Setter
@ToString
@NoArgsConstructor
@AllArgsConstructor
class Student {
    /** 姓名 **/
    private String name;

    /** 年龄 **/
    private int age;

    /** 性别 **/
    private Gender gender;

    /** 班级 **/
    private Grade grade;
}

enum Gender {
    MALE, FEMALE
}

enum Grade {
    ONE, TWO, THREE, FOUR;
}
```

### 经典案例

- 历史遗留代码

  ```java
  public static class Step0 implements LongTrackFinder {
      public Set<String> findLongTracks(List<Album> albums) {
          Set<String> trackNames = new HashSet<>();
          for (Album album : albums) {
              for (Track track : album.getTrackList()) {
                  if (track.getLength() > 60) {
                      String name = track.getName();
                      trackNames.add(name);
                  }
              }
          }
          return trackNames;
      }
  }
  ```

- 优化第一步

  ```java
  public static class Step1 implements LongTrackFinder {
      public Set<String> findLongTracks(List<Album> albums) {
          Set<String> trackNames = new HashSet<>();
          albums.stream().forEach(album -> album.getTracks().forEach(track -> {
              if (track.getLength() > 60) {
                  String name = track.getName();
                  trackNames.add(name);
              }
          }));
          return trackNames;
      }
  }
  ```

- 优化第二步

  ```java
  public static class Step2 implements LongTrackFinder {
      public Set<String> findLongTracks(List<Album> albums) {
          Set<String> trackNames = new HashSet<>();
          albums.stream().forEach(album -> album.getTracks().filter(track -> track.getLength() > 60).map(track -> track.getName()).forEach(name -> trackNames.add(name)));
          return trackNames;
      }
  }
  ```

- 优化第三步

  ```java
  public static class Step3 implements LongTrackFinder {
      public Set<String> findLongTracks(List<Album> albums) {
          Set<String> trackNames = new HashSet<>();
          albums.stream().flatMap(album -> album.getTracks()).filter(track -> track.getLength() > 60).map(track -> track.getName()).forEach(name -> trackNames.add(name));
          return trackNames;
      }
  }
  ```

- 优化第四步

  ```java
  public static class Step4 implements LongTrackFinder {
      public Set<String> findLongTracks(List<Album> albums) {
          return albums.stream().flatMap(album -> album.getTracks()).filter(track -> track.getLength() > 60).map(track -> track.getName()).collect(toSet());
      }
  }
  ```

### Eager&Lazy

```java
判断一个操作是惰性求值还是及早求值只需要看它的返回值
如果返回值是Stream那么是惰性求值，如果返回值是另一个值或为空，那么就是及早求值
```

### Stream运行机制

```reStructuredText
1.链式调用 一个元素只迭代一次
2.每一个中间操作返回一个新的流，流的属性sourceStage指向Head
3.Head->nextStage->nextStage...->null
4.有状态操作会把无状态操作截断单独处理 有状态操作:2个入参 无状态操作:1个入参
5.并行环境下有状态的中间操作不一定能并行操作
6.parrallel/sequetial也是中间操作(也是返回Stream)，它们不创建流只修改Head的并行标志
```

## 断点调试

```java
插件:Java Stream Debugger
```

