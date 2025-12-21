[toc]

## 一、收集器核心概念：Stream数据处理的终极武器

在Java 8的Stream API中，**收集器（Collector）** 是Stream流水线的终端操作，负责将流中的元素累积成一个最终结果。`Collectors`工具类提供了大量预定义的收集器实现，让我们能够以声明式的方式完成复杂的数据转换和聚合操作。

### 1.1 收集器的工作原理

收集器的核心机制基于四个组件函数，共同协作完成元素的累积过程：

- **Supplier（供应器）**：创建一个新的可变结果容器
- **Accumulator（累积器）**：将新元素合并到结果容器中
- **Combiner（组合器）**：将两个结果容器合并为一个（用于并行处理）
- **Finisher（完成器）**：对结果容器执行最终的转换（可选）

这种设计使得收集器既支持顺序处理，也天然支持并行处理，能够充分利用多核架构的优势。

### 1.2 收集器的三大核心功能

根据实际应用场景，收集器的主要功能可以分为三大类：

1. **数据收集**：将流元素收集到各种集合容器中（List、Set、Map等）
2. **聚合归约**：执行统计、求和、最值、平均等聚合计算
3. **前后处理**：实现分组、分区等复杂数据重组操作

## 二、归约与汇总操作实战

### 2.1 基础收集：toList、toSet、toCollection

最基本的收集操作是将流元素转换为集合，这是日常开发中最常见的场景。

```java
// 准备测试数据：产品列表
List<Product> products = Arrays.asList(
    new Product("Laptop", "Electronics", 999.99),
    new Product("Mouse", "Electronics", 29.99),
    new Product("Shirt", "Clothing", 49.99),
    new Product("Pants", "Clothing", 79.99),
    new Product("Laptop", "Electronics", 1299.99) // 重复商品，不同价格
);

// 转换为List（保留顺序和重复元素）
List<String> productNamesList = products.stream()
    .map(Product::getName)
    .collect(Collectors.toList());
System.out.println("产品名称列表: " + productNamesList);

// 转换为Set（自动去重）
Set<String> uniqueProductNames = products.stream()
    .map(Product::getName)
    .collect(Collectors.toSet());
System.out.println("唯一产品名称: " + uniqueProductNames);

// 转换为特定类型的集合（如TreeSet进行排序）
TreeSet<String> sortedProductNames = products.stream()
    .map(Product::getName)
    .collect(Collectors.toCollection(TreeSet::new));
System.out.println("排序后的产品名称: " + sortedProductNames);
```

### 2.2 统计汇总：counting、summing、averaging

对于数值型数据，Collectors提供了丰富的统计汇总方法。

```java
// 基础统计
long productCount = products.stream()
    .collect(Collectors.counting());
System.out.println("产品总数: " + productCount);

// 价格求和
double totalValue = products.stream()
    .collect(Collectors.summingDouble(Product::getPrice));
System.out.println("库存总价值: " + totalValue);

// 平均价格
double averagePrice = products.stream()
    .collect(Collectors.averagingDouble(Product::getPrice));
System.out.println("平均价格: " + averagePrice);

// 综合统计（一次获取所有统计信息）
DoubleSummaryStatistics stats = products.stream()
    .collect(Collectors.summarizingDouble(Product::getPrice));
System.out.println("价格统计: " + stats);
System.out.printf("统计详情: 数量=%d, 总和=%.2f, 平均=%.2f, 最小=%.2f, 最大=%.2f\n",
    stats.getCount(), stats.getSum(), stats.getAverage(), 
    stats.getMin(), stats.getMax());
```

### 2.3 字符串拼接：joining的巧妙运用

`joining`收集器专门用于字符串连接操作，特别适合生成CSV格式数据或日志信息。

```java
// 简单连接
String allProductNames = products.stream()
    .map(Product::getName)
    .collect(Collectors.joining());
System.out.println("连接结果: " + allProductNames);

// 带分隔符的连接
String productNamesCSV = products.stream()
    .map(Product::getName)
    .collect(Collectors.joining(", "));
System.out.println("CSV格式: " + productNamesCSV);

// 带前缀和后缀的连接（生成SQL IN语句）
String sqlInClause = products.stream()
    .map(Product::getName)
    .map(name -> "'" + name + "'")
    .collect(Collectors.joining(", ", "IN (", ")"));
System.out.println("SQL条件: " + sqlInClause);
```

## 三、分组操作：groupingBy的强大功能

分组是数据处理中最强大的功能之一，`groupingBy`让我们能够像SQL的GROUP BY一样对数据进行分组聚合。

### 3.1 基础分组

```java
// 按产品类别分组
Map<String, List<Product>> productsByCategory = products.stream()
    .collect(Collectors.groupingBy(Product::getCategory));
System.out.println("按类别分组: " + productsByCategory);

// 遍历分组结果
productsByCategory.forEach((category, productList) -> {
    System.out.println("类别: " + category + ", 产品数量: " + productList.size());
    productList.forEach(product -> 
        System.out.println("  - " + product.getName() + ": $" + product.getPrice()));
});
```

### 3.2 多级分组：实现复杂数据分析

多级分组类似于SQL中的多字段GROUP BY，可以实现更细粒度的数据划分。

```java
// 先按类别分组，再按价格区间分组
Map<String, Map<String, List<Product>>> multiLevelGrouping = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,  // 第一级分组：类别
        Collectors.groupingBy(product -> {  // 第二级分组：价格区间
            if (product.getPrice() < 50) return "低价";
            else if (product.getPrice() < 500) return "中价";
            else return "高价";
        })
    ));

// 输出多级分组结果
multiLevelGrouping.forEach((category, priceGroup) -> {
    System.out.println("\n类别: " + category);
    priceGroup.forEach((priceRange, productList) -> {
        System.out.println("  价格区间: " + priceRange + ", 产品数量: " + productList.size());
    });
});
```

### 3.3 分组后聚合计算

分组后经常需要对每个组进行聚合计算，这是`groupingBy`最强大的特性之一。

```java
// 按类别分组并计算每类的总价值
Map<String, Double> totalValueByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.summingDouble(Product::getPrice)
    ));
System.out.println("各类别总价值: " + totalValueByCategory);

// 按类别分组并统计每类的产品数量
Map<String, Long> productCountByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.counting()
    ));
System.out.println("各类别产品数量: " + productCountByCategory);

// 按类别分组并找出每类最贵的产品
Map<String, Optional<Product>> mostExpensiveByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.maxBy(Comparator.comparingDouble(Product::getPrice))
    ));
System.out.println("各类别最贵产品: ");
mostExpensiveByCategory.forEach((category, product) -> 
    product.ifPresent(p -> 
        System.out.println("  " + category + ": " + p.getName() + " - $" + p.getPrice())
    )
);
```

## 四、分区操作：partitioningBy的二分魔法

分区是分组的一种特殊形式，基于Predicate将数据分为**true和false两组**。

### 4.1 基础分区应用

```java
// 将产品分为高价和低价两类
Map<Boolean, List<Product>> partitionedProducts = products.stream()
    .collect(Collectors.partitioningBy(product -> product.getPrice() > 100));
System.out.println("高价产品: " + partitionedProducts.get(true));
System.out.println("低价产品: " + partitionedProducts.get(false));

// 分区并统计
Map<Boolean, Long> countByPartition = products.stream()
    .collect(Collectors.partitioningBy(
        product -> product.getPrice() > 100,
        Collectors.counting()
    ));
System.out.println("高价产品数量: " + countByPartition.get(true));
System.out.println("低价产品数量: " + countByPartition.get(false));
```

### 4.2 分区在业务逻辑中的应用

分区操作特别适合处理业务中的二分逻辑，如合格/不合格、已完成/未完成等场景。

```java
// 电商订单处理：区分已支付和未支付订单
List<Order> orders = getOrders(); // 获取订单列表

Map<Boolean, List<Order>> ordersByPaymentStatus = orders.stream()
    .collect(Collectors.partitioningBy(Order::isPaid));

// 对已支付订单进行进一步处理
List<Order> paidOrders = ordersByPaymentStatus.get(true);
double revenue = paidOrders.stream()
    .mapToDouble(Order::getTotalAmount)
    .sum();
System.out.println("总收入: $" + revenue);

// 对未支付订单发送提醒
List<Order> unpaidOrders = ordersByPaymentStatus.get(false);
unpaidOrders.forEach(order -> 
    System.out.println("发送支付提醒给: " + order.getCustomerEmail())
);
```

## 五、高级技巧：多级分组与自定义收集器

### 5.1 复杂的多级分组与聚合

结合多种收集器，可以实现极其强大的数据分析能力。

```java
// 复杂分析：按类别分组，然后分析每个类别的价格分布
Map<String, DoubleSummaryStatistics> categoryPriceStats = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.summarizingDouble(Product::getPrice)
    ));

categoryPriceStats.forEach((category, stats) -> {
    System.out.printf("类别%s: 平均价格$%.2f, 价格范围$%.2f-$%.2f, 产品数%d\n",
        category, stats.getAverage(), stats.getMin(), stats.getMax(), stats.getCount());
});

// 多级分组与映射结合：按类别分组，然后提取产品名称列表
Map<String, Set<String>> productNamesByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.mapping(Product::getName, Collectors.toSet())
    ));
System.out.println("各类别产品名称: " + productNamesByCategory);
```

### 5.2 自定义收集器解决特殊需求

虽然`Collectors`提供了丰富的预定义收集器，但有时我们需要解决特殊业务需求。

```java
// 自定义收集器：收集价格最高的前N个产品
public static <T> Collector<T, ?, List<T>> topNCollector(int n, Comparator<T> comparator) {
    return Collector.of(
        () -> new PriorityQueue<>(comparator), // Supplier
        (queue, element) -> { // Accumulator
            queue.offer(element);
            if (queue.size() > n) {
                queue.poll(); // 移除最小的元素
            }
        },
        (queue1, queue2) -> { // Combiner
            queue2.forEach(queue1::offer);
            while (queue1.size() > n) {
                queue1.poll();
            }
            return queue1;
        },
        queue -> { // Finisher
            List<T> result = new ArrayList<>(queue);
            result.sort(comparator.reversed());
            return result;
        }
    );
}

// 使用自定义收集器获取价格最高的3个产品
List<Product> top3ExpensiveProducts = products.stream()
    .collect(topNCollector(3, Comparator.comparingDouble(Product::getPrice)));
System.out.println("价格最高的3个产品: " + top3ExpensiveProducts);
```

## 六、性能优化与最佳实践

### 6.1 选择正确的收集策略

不同的收集器在性能上有显著差异，特别是在处理大数据集时。

```java
// 性能对比：普通toList vs 预分配大小的toCollection
List<Product> largeProductList = generateLargeProductList(100000);

// 方式1：使用toList（可能涉及多次扩容）
long startTime = System.currentTimeMillis();
List<String> names1 = largeProductList.stream()
    .map(Product::getName)
    .collect(Collectors.toList());
long time1 = System.currentTimeMillis() - startTime;

// 方式2：使用toCollection预分配大小（性能更优）
startTime = System.currentTimeMillis();
List<String> names2 = largeProductList.stream()
    .map(Product::getName)
    .collect(Collectors.toCollection(() -> new ArrayList<>(largeProductList.size())));
long time2 = System.currentTimeMillis() - startTime;

System.out.printf("toList耗时: %dms, toCollection耗时: %dms\n", time1, time2);
```

### 6.2 并行流中的收集器使用

大多数Collectors都支持并行处理，但需要注意线程安全问题。

```java
// 并行流中的分组操作
Map<String, List<Product>> parallelGrouping = products.parallelStream()
    .collect(Collectors.groupingByConcurrent(Product::getCategory));
System.out.println("并行分组结果: " + parallelGrouping);

// 并行求和
double parallelTotal = products.parallelStream()
    .collect(Collectors.summingDouble(Product::getPrice));
System.out.println("并行计算总值: " + parallelTotal);
```

## 七、综合实战案例：电商数据分析系统

让我们通过一个完整的电商数据分析案例，综合运用各种收集器技术。

```java
public class ECommerceAnalytics {
    
    public void analyzeSalesData(List<Order> orders) {
        // 1. 销售总额分析
        DoubleSummaryStatistics salesStats = orders.stream()
            .filter(Order::isCompleted)
            .collect(Collectors.summarizingDouble(Order::getTotalAmount));
        
        // 2. 按客户分组分析
        Map<Customer, DoubleSummaryStatistics> customerStats = orders.stream()
            .collect(Collectors.groupingBy(
                Order::getCustomer,
                Collectors.summarizingDouble(Order::getTotalAmount)
            ));
        
        // 3. 按时间分区（本月 vs 历史）
        LocalDate now = LocalDate.now();
        Map<Boolean, List<Order>> ordersByRecency = orders.stream()
            .collect(Collectors.partitioningBy(
                order -> order.getOrderDate().getMonth() == now.getMonth()
            ));
        
        // 4. 多级分析：按类别->价格区间分组
        Map<String, Map<String, List<Product>>> categoryPriceAnalysis = 
            orders.stream()
                .flatMap(order -> order.getItems().stream())
                .collect(Collectors.groupingBy(
                    item -> item.getProduct().getCategory(),
                    Collectors.groupingBy(item -> {
                        double price = item.getProduct().getPrice();
                        if (price < 50) return "经济型";
                        else if (price < 200) return "标准型";
                        else return "高端型";
                    })
                ));
        
        // 输出分析报告
        generateReport(salesStats, customerStats, ordersByRecency, categoryPriceAnalysis);
    }
    
    private void generateReport(DoubleSummaryStatistics salesStats,
                              Map<Customer, DoubleSummaryStatistics> customerStats,
                              Map<Boolean, List<Order>> ordersByRecency,
                              Map<String, Map<String, List<Product>>> categoryPriceAnalysis) {
        System.out.println("=== 电商销售分析报告 ===");
        System.out.printf("总销售额: $%.2f\n", salesStats.getSum());
        System.out.printf("平均订单额: $%.2f\n", salesStats.getAverage());
        System.out.printf("最大订单额: $%.2f\n", salesStats.getMax());
        
        System.out.println("\n=== 客户价值分析 ===");
        customerStats.entrySet().stream()
            .sorted((e1, e2) -> Double.compare(e2.getValue().getSum(), e1.getValue().getSum()))
            .limit(5)
            .forEach(entry -> System.out.printf("客户%s: 总消费$%.2f, 订单数%d\n",
                entry.getKey().getName(), entry.getValue().getSum(), entry.getValue().getCount()));
        
        System.out.println("\n=== 产品类别分析 ===");
        categoryPriceAnalysis.forEach((category, priceGroups) -> {
            System.out.println("类别: " + category);
            priceGroups.forEach((priceRange, products) -> {
                System.out.printf("  %s: %d个产品\n", priceRange, products.size());
            });
        });
    }
}
```

## 八、总结

Collectors是Java 8 Stream API的精华所在，它让复杂的数据聚合操作变得简单而优雅。通过本讲的学习，我们掌握了：

1. **基础收集**：`toList`、`toSet`、`toCollection`等基本收集操作
2. **统计汇总**：`counting`、`summing`、`averaging`等数值聚合方法
3. **分组操作**：`groupingBy`的单级和多级分组能力
4. **分区操作**：`partitioningBy`的二分数据处理
5. **高级技巧**：多级分组聚合和自定义收集器

**关键实践要点**：
- 根据数据规模和性能要求选择合适的收集策略
- 利用多级分组实现复杂的数据分析
- 在并行处理场景下注意线程安全问题
- 结合`mapping`、`filtering`等下游收集器实现更灵活的数据处理

Collectors的强大之处在于它们的**可组合性**，通过将简单的收集器组合起来，可以构建出处理复杂数据转换的强大流水线。掌握这些技术后，你会发现以前需要几十行代码才能完成的数据处理任务，现在只需要几行声明式的代码就能优雅解决。

**思考挑战**：在你的当前项目中，哪些数据处理场景可以用Collectors进行重构？尝试用本讲介绍的技术优化一个复杂的数据处理逻辑，体验代码简洁性和性能的双重提升！

**下期预告：第9讲：Stream并行流与性能——利用多核架构**

---

更多技术干货欢迎关注微信公众号“**科威舟的AI笔记**”~

![](https://files.mdnice.com/user/101007/5b1be244-b402-456b-bafb-63490ab66749.jpg)

【转载须知】：<font color=red>**转载请注明原文出处及作者信息**</font>