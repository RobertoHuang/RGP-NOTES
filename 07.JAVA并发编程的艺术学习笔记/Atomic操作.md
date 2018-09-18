- 原子更新基本类型

  - `AtomicBoolean`
  - `AtomicInteger`
  - `AtomicLong`

  以上3个类提供的方法几乎一模一样，以`AtomicInteger`为例进行详解

  ```java
  // 以原子的方式将输入的数值与实例中的值相加，并返回结果
  int addAndGet(int delta) 
  // 如果输入的值等于预期值，则以原子方式将该值设置为输入的值
  boolean compareAndSet(int expect, int update) 
  // 以原子的方式将当前值加1，注意这里返回的是自增前的值
  int getAndIncrement()
  // 最终会设置成newValue,使用lazySet设置值后可能导致其他线程在之后的一小段时间内还是可以读到旧的值
  void lazySet(int newValue)
  // 以原子的方式设置为newValue并返回旧值
  int getAndSet(int newValue)
  ```

- 原子更新数组

    - `AtomicIntegerArray`
    - `AtomicLongArray`
    - `AtomicReferenceArray`

  ```java
  // 获取索引为index的元素值
  get(int index)
  // 如果当前值等于预期值，则以原子方式将数组位置i的元素设置为update值
  compareAndSet(int i,E expect,E update)
  ```

- 原子更新字段类

  - `AtomicIntegerFieldUpdater`
  - `AtomicLongFieldUpdater`
  - `AtomicStampedFieldUpdater`
  - `AtomicReferenceFieldUpdater`

- 原子更新引用类型

  - `AtomicReference`
  - `AtomicReferenceFieldUpdater`
  - `AtomicMarkableReference`
