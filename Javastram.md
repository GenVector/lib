##1、new Random(47).ints(int1, int2) 产生stream
##2、Stream.of(object1,object2...)将一组元素产生stream
##3、集合类stream()方法产生流;MAP.entrySet().stream()
##4、Stream.generate() 的用法，它可以把任意 Supplier<T> 用于生成 T 类型的流。
	属于function系列接口规范
    需要注意的是,谨慎使用parallel。generate()创建出来的是一个无限流,可能你只需要其中的N个,在parallel的情况下实际上初始化的对象远远不止这个数量级。
    默认并行:512
###查一下

##5、collect()收集操作，它根据参数来组合所有流中的元素。
##6、NIO:Files.readAllLines(Paths.get(fname))
##7、IntStream.range(int1,int2)产生连续的IntStream,可用于for循环
##8、iterate()
	Stream.iterate(0, i -> {int result = x + i;x = i;return result;});
##9、建造者模式
	Stream.Builder 调用 build() 方法后继续尝试添加会产生一个异常
	public class FileToWordsBuilder {
    Stream.Builder<String> builder = Stream.builder();
    
    public FileToWordsBuilder(String filePath) throws Exception {
        Files.lines(Paths.get(filePath)).skip(1).forEach(line -> {
                  for (String w : line.split("[ .?,]+"))
                      builder.add(w);
              });
    }
    
    Stream<String> stream() {
        return builder.build();
    }
    
    public static void main(String[] args) throws Exception {
        new FileToWordsBuilder("Cheese.dat")
            .stream()
            .limit(7)
            .map(w -> w + " ")
            .forEach(System.out::print);
    }
}
##10、Arrays.stream()将数组产生一个stream
##11、Java8在java.util.regex.Pattern 中增加了一个新的方法 splitAsStream()。这个方法可以根据传入的公式将字符序列转化为流。但是有一个限制,输入只能是 CharSequence
##12、String.CASE_INSENSITIVE_ORDER	比较器:根据ASCII大小
##13、IntStream.concat   以参数顺序组合两个流
##13、中间操作
    stream.map()用于类型转换,操作元素
    peek() 操作的目的是帮助调试。它允许你无修改地查看流中的元素

##14、移除元素
    distinct():在 Randoms.java 类中的 distinct() 可用于消除流中的重复元素。相比创建一个 Set 集合，该方法的工作量要少得多。

    filter(Predicate):过滤操作会保留与传递进去的过滤器函数计算结果为 true 元素。
##15、MAP应用函数到元素
    map(Function)：将函数操作应用在输入流的元素中，并将返回值传递到输出流中。

    mapToInt(ToIntFunction)：操作同上，但结果是 IntStream。

    mapToLong(ToLongFunction)：操作同上，但结果是 LongStream。

mapToDouble(ToDoubleFunction)：操作同上，但结果是 DoubleStream。

##16、Optional流
Stream.generate(Signal::morse)
                .map(signal -> Optional.ofNullable(signal));

##17、终端操作
    toArray()：将流转换成适当类型的数组。

    toArray(generator)：在特殊情况下，生成器用于分配自定义的数组存储。

    forEach(Consumer)：常见的,如 System.out::println 作为 Consumer 函数。

    forEachOrdered(Consumer)： 保证 forEach 按照原始流顺序操作。

        第一种形式：显式设计为任意顺序操作元素，仅在引入 parallel() 操作时才有意义。这里简单介绍下parallel()：可实现多处理器并行操作。实现原理为将流分割为多个（通常数目为 CPU 核心数）并在不同处理器上分别执行操作。这对内部循环是可行的。
##18、parallel!!
    parallel产生并行流,并行处理。
    parallel并不是灵丹妙药,很多时候会出现意想不到的状况
##收集
    collect(Collector)：使用 Collector 收集流元素到结果集合中。
        Collectors.toCollection()
    collect(Supplier, BiConsumer, BiConsumer)
##元素查找
    findFirst()：返回一个含有第一个流元素的 Optional，如果流为空返回 Optional.empty。
    findAny(：返回含有任意流元素的 Optional，如果流为空返回 Optional.empty。
##12、count()：流中的元素个数
	max(Comparator)：根据所传入的 Comparator 所决定的“最大”元素。
	min(Comparator)
	String常用的比较器

##13、一些数字流操作
	average() ：求取流元素平均值。
	max() 和 min()：因为这些操作在数字流上面，所以不需要 Comparator。
	sum()：对所有流元素进行求和。
	summaryStatistics()：生成可能有用的数据。目前还不太清楚他们为什么觉得有必要这样做，但是你可以直接使用方法产生所有的数据。
##方法引用:参数不一致的传递问题

##元素匹配
    1、allMatch(Predicate) ：如果流的每个元素根据提供的 Predicate 都返回 true 时，结果返回为 true。这个操作将会在第一个 false 之后短路；也就是不会在发生 false 之后继续执行计算。
    2、anyMatch(Predicate)：如果流中的一个元素根据提供的 Predicate 返回 true 时，结果返回为 true。这个操作将会在第一个 true 之后短路；也就是不会在发生 true 之后继续执行计算。
    3、noneMatch(Predicate)：如果流的每个元素根据提供的 Predicate 都返回 false 时，结果返回为 true。这个操作将会在第一个 true 之后短路；也就是不会在发生 true 之后继续执行计算。
###    基于上个标题的理解


##MAP、FOREACH、PEEK
