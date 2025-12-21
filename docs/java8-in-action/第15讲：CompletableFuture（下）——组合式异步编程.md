# 第15讲：CompletableFuture（下）——组合式异步编程

## 一、组合式异步编程的核心价值

在现代分布式系统和微服务架构中，**组合式异步编程**已成为提升应用性能的关键技术。传统的同步阻塞编程模式在面对多个远程服务调用时，往往导致线程长时间等待，造成资源浪费和响应延迟。而CompletableFuture提供的组合能力，让我们能够像搭积木一样将多个异步任务优雅地组合起来，充分发挥多核处理器和并行计算的优势。

**组合式编程 vs 传统编程对比**：
- **串行阻塞模式**：总耗时 = 各任务耗时之和
- **并行异步模式**：总耗时 ≈ 最慢任务的耗时

这种模式转变带来的性能提升在I/O密集型场景中尤为显著，如下图所示的任务依赖关系：

```
服务A调用(200ms)  服务B调用(150ms)  服务C调用(100ms)
       ↘             ↓             ↙
       聚合处理(50ms) → 最终结果(总耗时≈200ms而非500ms)
```

## 二、thenCompose：扁平化链式依赖任务

### 2.1 原理深度解析

`thenCompose`方法的核心作用是**解决回调地狱**（Callback Hell）问题。当多个异步任务存在**顺序依赖关系**时（即后一个任务需要前一个任务的结果作为输入），`thenCompose`能够将嵌套的CompletableFuture扁平化。

**传统嵌套方式的问题**：
```java
// 回调地狱示例：可读性差且难以维护
CompletableFuture<CompletableFuture<CompletableFuture<String>>> nestedFuture = 
    getUserInfo(userId)
        .thenApply(user -> getUserCredit(user.getId()))
        .thenApply(credit -> generateReport(credit.getScore()));
```

**使用thenCompose的优化方案**：
```java
CompletableFuture<String> flatFuture = 
    getUserInfo(userId)
        .thenCompose(user -> getUserCredit(user.getId()))
        .thenCompose(credit -> generateReport(credit.getScore()));
```

### 2.2 实战案例：用户订单处理流水线

```java
public class OrderProcessingService {
    
    // 模拟获取用户基本信息
    private CompletableFuture<User> getUserInfo(Long userId) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("获取用户信息: " + userId);
            sleep(100); // 模拟网络延迟
            return new User(userId, "用户" + userId);
        });
    }
    
    // 模拟获取用户订单历史（依赖用户信息）
    private CompletableFuture<List<Order>> getOrderHistory(User user) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("获取用户" + user.getId() + "的订单历史");
            sleep(150);
            return Arrays.asList(
                new Order(1L, user.getId(), 299.99),
                new Order(2L, user.getId(), 159.50)
            );
        });
    }
    
    // 模拟生成用户报告（依赖订单历史）
    private CompletableFuture<String> generateUserReport(List<Order> orders) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("生成用户报告，订单数量: " + orders.size());
            sleep(200);
            double totalAmount = orders.stream().mapToDouble(Order::getAmount).sum();
            return String.format("用户消费报告: 总订单数%d, 总金额%.2f", 
                               orders.size(), totalAmount);
        });
    }
    
    // 使用thenCompose构建处理流水线
    public CompletableFuture<String> processUserOrderPipeline(Long userId) {
        return getUserInfo(userId)
            .thenCompose(this::getOrderHistory)      // 扁平化处理
            .thenCompose(this::generateUserReport)    // 继续扁平化
            .exceptionally(ex -> {
                System.err.println("处理流水线失败: " + ex.getMessage());
                return "报告生成失败";
            });
    }
    
    // 使用示例
    public static void main(String[] args) throws Exception {
        OrderProcessingService service = new OrderProcessingService();
        CompletableFuture<String> reportFuture = service.processUserOrderPipeline(1001L);
        
        // 异步获取结果
        reportFuture.thenAccept(report -> 
            System.out.println("最终报告: " + report)
        );
        
        // 主线程继续执行其他任务
        System.out.println("主线程继续执行...");
        Thread.sleep(1000); // 等待异步任务完成
    }
    
    private static void sleep(long millis) {
        try { Thread.sleep(millis); } 
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

## 三、thenCombine：合并并行独立任务

### 3.1 原理深度解析

`thenCombine`用于合并**两个并行执行且相互独立**的异步任务的结果。当两个任务之间没有依赖关系，但后续处理需要同时使用它们的结果时，`thenCombine`是最佳选择。

**方法签名分析**：
```java
<U,V> CompletableFuture<V> thenCombine(
    CompletionStage<? extends U> other,
    BiFunction<? super T,? super U,? extends V> fn
)
```
- `other`：另一个CompletionStage（并行任务）
- `fn`：合并函数，接收两个任务的结果并返回合并后的结果

### 3.2 实战案例：电商价格计算服务

```java
public class PriceCalculationService {
    
    // 并行获取商品基础价格和促销信息
    private CompletableFuture<Double> getBasePrice(Long productId) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("查询商品基础价格: " + productId);
            sleep(120);
            return 299.0; // 模拟基础价格
        });
    }
    
    private CompletableFuture<Double> getPromotionDiscount(Long productId) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("查询促销折扣: " + productId);
            sleep(100);
            return 0.15; // 模拟85折优惠
        });
    }
    
    private CompletableFuture<Double> calculateShippingFee(String region) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("计算运费: " + region);
            sleep(80);
            return "北京".equals(region) ? 0.0 : 15.0; // 北京免运费
        });
    }
    
    // 使用thenCombine合并价格和折扣
    public CompletableFuture<Double> calculateFinalPrice(Long productId, String region) {
        CompletableFuture<Double> basePriceFuture = getBasePrice(productId);
        CompletableFuture<Double> discountFuture = getPromotionDiscount(productId);
        CompletableFuture<Double> shippingFeeFuture = calculateShippingFee(region);
        
        // 合并基础价格和折扣
        CompletableFuture<Double> priceAfterDiscount = basePriceFuture
            .thenCombine(discountFuture, (price, discount) -> {
                System.out.printf("合并计算: 价格%.2f * 折扣%.2f%n", price, (1 - discount));
                return price * (1 - discount);
            });
        
        // 再次合并运费
        return priceAfterDiscount.thenCombine(shippingFeeFuture, (price, shipping) -> {
            System.out.printf("最终计算: 折后价%.2f + 运费%.2f%n", price, shipping);
            double finalPrice = price + shipping;
            System.out.println("最终价格: " + finalPrice);
            return finalPrice;
        });
    }
    
    // 复杂合并：多个thenCombine链式调用
    public CompletableFuture<String> getProductDetail(Long productId) {
        CompletableFuture<String> basicInfo = getBasicInfo(productId);
        CompletableFuture<String> inventoryInfo = getInventoryInfo(productId);
        CompletableFuture<String> reviewInfo = getReviewInfo(productId);
        
        return basicInfo
            .thenCombine(inventoryInfo, (basic, inventory) -> 
                basic + " | " + inventory)
            .thenCombine(reviewInfo, (combined, review) -> 
                combined + " | " + review);
    }
}
```

## 四、allOf/anyOf：多任务协调控制

### 4.1 allOf - 等待所有任务完成

`allOf`用于等待**多个异步任务全部完成**，适用于需要聚合多个独立任务结果的场景。

```java
public class BatchProcessingService {
    
    // 模拟多个微服务调用
    private CompletableFuture<String> serviceA() {
        return CompletableFuture.supplyAsync(() -> { sleep(200); return "服务A结果"; });
    }
    
    private CompletableFuture<String> serviceB() {
        return CompletableFuture.supplyAsync(() -> { sleep(150); return "服务B结果"; });
    }
    
    private CompletableFuture<String> serviceC() {
        return CompletableFuture.supplyAsync(() -> { sleep(180); return "服务C结果"; });
    }
    
    // 使用allOf等待所有服务完成
    public CompletableFuture<List<String>> aggregateAllServices() {
        CompletableFuture<String> futureA = serviceA();
        CompletableFuture<String> futureB = serviceB();
        CompletableFuture<String> futureC = serviceC();
        
        // 等待所有任务完成
        CompletableFuture<Void> allFutures = CompletableFuture.allOf(futureA, futureB, futureC);
        
        // 所有任务完成后提取各自结果
        return allFutures.thenApply(v -> {
            List<String> results = new ArrayList<>();
            // 注意：这里的join()不会阻塞，因为任务已经完成
            results.add(futureA.join());
            results.add(futureB.join());
            results.add(futureC.join());
            return results;
        });
    }
    
    // 优化版：使用Stream处理动态数量的任务
    public CompletableFuture<List<String>> processBatch(List<Long> ids) {
        List<CompletableFuture<String>> futures = ids.stream()
            .map(id -> processSingleItem(id))
            .collect(Collectors.toList());
        
        CompletableFuture<Void> allDone = CompletableFuture.allOf(
            futures.toArray(new CompletableFuture[0])
        );
        
        return allDone.thenApply(v -> futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList()));
    }
    
    private CompletableFuture<String> processSingleItem(Long id) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(100);
            return "处理结果:" + id;
        });
    }
}
```

### 4.2 anyOf - 竞速获取最快结果

`anyOf`用于获取**多个任务中最快完成的结果**，适用于冗余服务调用或超时降级场景。

```java
public class FastestResponseService {
    
    // 多个同质服务，返回最快的结果
    public CompletableFuture<String> getFastestResponse(String request) {
        CompletableFuture<String> primaryService = callPrimaryService(request);
        CompletableFuture<String> secondaryService = callSecondaryService(request);
        CompletableFuture<String> cacheService = getFromCache(request);
        
        return CompletableFuture.anyOf(primaryService, secondaryService, cacheService)
            .thenApply(result -> (String) result);
    }
    
    // 带超时和降级的竞速调用
    public CompletableFuture<String> getWithTimeoutFallback(String request) {
        CompletableFuture<String> mainService = callMainService(request);
        CompletableFuture<String> fallbackService = callFallbackService(request);
        
        // 主服务超时设置
        CompletableFuture<String> timeoutFuture = CompletableFuture.supplyAsync(() -> {
            sleep(500); // 超时时间
            return "超时降级结果";
        });
        
        return CompletableFuture.anyOf(mainService, timeoutFuture, fallbackService)
            .thenApply(result -> {
                String response = (String) result;
                if ("超时降级结果".equals(response)) {
                    System.out.println("主服务超时，使用降级结果");
                    // 取消主服务调用以避免资源浪费
                    mainService.cancel(true);
                }
                return response;
            });
    }
}
```

## 五、综合实战：微服务聚合网关

下面通过一个完整的微服务聚合案例，展示CompletableFuture组合操作在真实业务场景中的应用。

### 5.1 业务场景描述

假设我们需要构建一个电商用户仪表板服务，需要聚合以下数据：
1. **用户基本信息服务**（核心数据）
2. **订单历史服务**（交易数据）
3. **推荐商品服务**（个性化推荐）
4. **用户积分服务**（忠诚度数据）

### 5.2 服务聚合实现

```java
public class UserDashboardAggregator {
    private UserService userService;
    private OrderService orderService;
    private RecommendationService recommendationService;
    private LoyaltyService loyaltyService;
    
    // 自定义线程池优化
    private ExecutorService ioBoundExecutor = Executors.newFixedThreadPool(10);
    
    public CompletableFuture<UserDashboard> aggregateUserDashboard(Long userId) {
        long startTime = System.currentTimeMillis();
        
        // 并行调用所有微服务
        CompletableFuture<UserInfo> userFuture = userService.getUserInfoAsync(userId);
        CompletableFuture<List<Order>> ordersFuture = orderService.getOrderHistoryAsync(userId);
        CompletableFuture<List<Product>> recommendationsFuture = 
            recommendationService.getPersonalizedRecommendationsAsync(userId);
        CompletableFuture<LoyaltyPoints> loyaltyFuture = loyaltyService.getLoyaltyPointsAsync(userId);
        
        // 使用allOf等待所有服务响应
        CompletableFuture<Void> allFutures = CompletableFuture.allOf(
            userFuture, ordersFuture, recommendationsFuture, loyaltyFuture
        );
        
        // 聚合所有结果
        CompletableFuture<UserDashboard> dashboardFuture = allFutures.thenApplyAsync(v -> {
            try {
                UserInfo user = userFuture.get();
                List<Order> orders = ordersFuture.get();
                List<Product> recommendations = recommendationsFuture.get();
                LoyaltyPoints loyalty = loyaltyFuture.get();
                
                UserDashboard dashboard = new UserDashboard();
                dashboard.setUser(user);
                dashboard.setRecentOrders(orders);
                dashboard.setRecommendations(recommendations);
                dashboard.setLoyaltyPoints(loyalty);
                dashboard.setLastUpdated(new Date());
                
                long duration = System.currentTimeMillis() - startTime;
                System.out.println("仪表板聚合完成，耗时: " + duration + "ms");
                
                return dashboard;
            } catch (Exception e) {
                throw new CompletionException("聚合仪表板失败", e);
            }
        }, ioBoundExecutor);
        
        // 异常处理和降级
        return dashboardFuture.exceptionally(ex -> {
            System.err.println("获取用户仪表板失败: " + ex.getMessage());
            return createFallbackDashboard(userId);
        });
    }
    
    // 复杂聚合：有依赖关系的服务调用
    public CompletableFuture<EnhancedDashboard> getEnhancedDashboard(Long userId) {
        // 先获取用户基本信息
        return userService.getUserInfoAsync(userId)
            .thenCompose(userInfo -> {
                // 并行获取依赖用户信息的其他数据
                CompletableFuture<List<Order>> ordersFuture = 
                    orderService.getOrderHistoryAsync(userId);
                CompletableFuture<List<Product>> recommendationsFuture = 
                    recommendationService.getRecommendationsByInterest(
                        userInfo.getInterests());
                CompletableFuture<LoyaltyPoints> loyaltyFuture = 
                    loyaltyService.getLoyaltyPointsAsync(userId);
                
                return CompletableFuture.allOf(ordersFuture, recommendationsFuture, loyaltyFuture)
                    .thenApply(v -> {
                        EnhancedDashboard dashboard = new EnhancedDashboard();
                        dashboard.setUserInfo(userInfo);
                        dashboard.setOrders(ordersFuture.join());
                        dashboard.setRecommendations(recommendationsFuture.join());
                        dashboard.setLoyaltyPoints(loyaltyFuture.join());
                        return dashboard;
                    });
            });
    }
}
```

### 5.3 性能监控和优化

```java
public class PerformanceMonitor {
    
    public static <T> CompletableFuture<T> monitorAsyncOperation(
        String operationName, 
        Supplier<CompletableFuture<T>> operation) {
        
        long startTime = System.currentTimeMillis();
        System.out.println("开始异步操作: " + operationName);
        
        return operation.get().whenComplete((result, ex) -> {
            long duration = System.currentTimeMillis() - startTime;
            if (ex == null) {
                System.out.println("操作 " + operationName + " 完成，耗时: " + duration + "ms");
            } else {
                System.err.println("操作 " + operationName + " 失败，耗时: " + duration + "ms");
            }
        });
    }
    
    // 使用示例
    public CompletableFuture<UserDashboard> getDashboardWithMonitoring(Long userId) {
        return monitorAsyncOperation("用户仪表板聚合", 
            () -> aggregateUserDashboard(userId));
    }
}
```

## 六、最佳实践与性能优化

### 6.1 线程池策略选择

根据任务特性选择合适的线程池至关重要：

```java
public class ThreadPoolStrategy {
    
    // I/O密集型任务：线程数 = CPU核心数 * (1 + 等待时间/计算时间)
    private ExecutorService ioIntensivePool = new ThreadPoolExecutor(
        20, 50, 60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(1000),
        new ThreadFactoryBuilder().setNameFormat("io-pool-%d").build()
    );
    
    // CPU密集型任务：线程数 ≈ CPU核心数
    private ExecutorService cpuIntensivePool = Executors.newWorkStealingPool();
    
    // 专用阻塞操作线程池
    private ExecutorService blockingPool = Executors.newFixedThreadPool(100);
    
    public CompletableFuture<String> processWithOptimalPool(String input) {
        // 第一阶段：I/O密集型
        return CompletableFuture.supplyAsync(() -> fetchData(input), ioIntensivePool)
            // 第二阶段：CPU密集型转换
            .thenApplyAsync(data -> transformData(data), cpuIntensivePool)
            // 第三阶段：可能阻塞的操作
            .thenApplyAsync(result -> blockingOperation(result), blockingPool);
    }
}
```

### 6.2 异常处理策略

```java
public class RobustComposition {
    
    public CompletableFuture<Result> robustPipeline(Input input) {
        return firstAsyncOperation(input)
            .thenCompose(this::secondAsyncOperation)
            .thenCombine(parallelOperation(), this::combineResults)
            .handle((result, ex) -> {
                if (ex != null) {
                    // 分级降级策略
                    if (ex instanceof TimeoutException) {
                        return getCachedResult();
                    } else if (ex instanceof ServiceUnavailableException) {
                        return getDefaultResult();
                    } else {
                        throw new CompletionException(ex);
                    }
                }
                return result;
            });
    }
    
    // 超时控制
    public CompletableFuture<String> withTimeout(CompletableFuture<String> future) {
        return future.orTimeout(5, TimeUnit.SECONDS)
            .exceptionally(ex -> {
                if (ex.getCause() instanceof TimeoutException) {
                    return "超时默认值";
                }
                throw new CompletionException(ex);
            });
    }
}
```

## 总结

通过本讲的学习，我们深入掌握了CompletableFuture的组合式异步编程能力。关键要点总结：

1. **`thenCompose`用于链式依赖**：解决回调地狱，扁平化嵌套的异步任务
2. **`thenCombine`用于并行合并**：合并独立任务的结果，提高并行效率
3. **`allOf`协调多个任务**：等待所有任务完成进行聚合处理
4. **`anyOf`竞速获取结果**：适用于冗余服务和降级场景

**核心价值**：组合式异步编程将复杂的异步流程转化为声明式的管道操作，大幅提升代码可读性和系统性能。在微服务架构和分布式系统中，这种编程模式能够将串行调用的耗时从各服务耗时之和降低到最慢服务的耗时水平。

**实践建议**：在实际项目中，建议根据任务特性选择合适的线程池，合理设置超时控制，并实现完善的异常处理机制。通过性能监控和优化，充分发挥CompletableFuture组合编程的威力。