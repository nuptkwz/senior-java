[toc]

大家好，我是你们的船长：**科威舟**，今天给大家分享一下：第14讲：CompletableFuture（上）——构建异步应用。更多技术干货欢迎关注微信公众号**科威舟的AI笔记**~

![](https://files.mdnice.com/user/101007/5b1be244-b402-456b-bafb-63490ab66749.jpg)

---

## 一、从Future到CompletableFuture：异步编程的演进

在Java 5中引入的`Future`接口为异步编程提供了基础支持，但它存在明显的局限性。`Future`代表一个异步计算的结果，允许我们提交任务到线程池，并在需要时获取计算结果。然而，在实际使用中，`Future`的模式显得笨重且功能有限。

### 1.1 Future的局限性

**传统Future的使用模式**：
```java
ExecutorService executor = Executors.newFixedThreadPool(5);
Future<String> future = executor.submit(new Callable<String>() {
    @Override
    public String call() throws Exception {
        Thread.sleep(2000); // 模拟耗时操作
        return "任务结果";
    }
});

// 阻塞等待结果
try {
    String result = future.get(5, TimeUnit.SECONDS);
    System.out.println("结果: " + result);
} catch (TimeoutException e) {
    System.out.println("任务超时");
} finally {
    executor.shutdown();
}
```

**Future的主要局限性**：
- **结果获取阻塞**：`get()`方法会阻塞线程直到结果可用
- **无法手动完成**：不能主动设置任务结果或异常
- **缺乏回调机制**：任务完成后无法自动触发后续操作
- **组合能力弱**：难以将多个异步任务的结果组合在一起
- **异常处理不便**：异常被包裹在ExecutionException中，处理繁琐

### 1.2 CompletableFuture的设计理念

Java 8引入的`CompletableFuture`解决了上述所有问题，它实现了`Future`和`CompletionStage`接口，提供了强大的异步编程能力。

**核心设计特点**：
- **可完成性**：可以手动设置结果或异常
- **非阻塞回调**：支持任务完成后的回调处理
- **链式编程**：支持多个异步任务的流水线组合
- **异常传播**：提供完整的异常处理机制
- **组合操作**：支持多个异步任务的并行组合

## 二、CompletableFuture核心原理解析

### 2.1 架构设计：观察者模式的应用

`CompletableFuture`内部采用**观察者模式**实现异步回调机制。每个`CompletableFuture`维护一个依赖动作栈（stack），当任务完成时，会自动通知所有注册的依赖动作。

**内部关键字段**：
```java
// 存储异步计算的结果
volatile Object result;

// 依赖动作栈（观察者链）
volatile Completion stack;
```

### 2.2 任务触发机制：tryFire方法

`CompletableFuture`通过`tryFire`方法实现异步任务链的触发。当某个阶段完成时，会递归调用后续阶段的`tryFire`方法，形成完整的任务执行链。

**触发模式**：
- **SYNC**（同步触发）：在当前线程直接执行
- **ASYNC**（异步触发）：提交到线程池异步执行
- **NESTED**（嵌套触发）：避免重复触发的特殊模式

## 三、创建异步任务：supplyAsync与runAsync

### 3.1 supplyAsync：有返回值的异步任务

`supplyAsync`方法接受一个`Supplier`函数式接口，返回一个`CompletableFuture`对象，用于执行有返回值的异步任务。

**基础用法**：
```java
public class SupplyAsyncExample {
    public static void main(String[] args) throws Exception {
        // 使用默认线程池（ForkJoinPool.commonPool()）
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            // 模拟耗时操作
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new IllegalStateException(e);
            }
            return "异步任务结果";
        });
        
        System.out.println("主线程继续执行...");
        String result = future.get();
        System.out.println("获取结果: " + result);
    }
}
```

**使用自定义线程池**：
```java
public class CustomThreadPoolExample {
    // 创建专用的线程池
    private static final ExecutorService customExecutor = 
        Executors.newFixedThreadPool(5);
    
    public static void main(String[] args) throws Exception {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("执行线程: " + Thread.currentThread().getName());
            return "使用自定义线程池";
        }, customExecutor);
        
        System.out.println("结果: " + future.get());
        customExecutor.shutdown();
    }
}
```

### 3.2 runAsync：无返回值的异步任务

`runAsync`方法接受一个`Runnable`参数，适用于不需要返回值的异步操作。

**实战示例**：
```java
public class RunAsyncExample {
    public static void main(String[] args) throws Exception {
        CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
            // 执行后台任务，如日志记录、数据清理等
            System.out.println("开始执行后台任务...");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                throw new IllegalStateException(e);
            }
            System.out.println("后台任务执行完成");
        });
        
        // 等待任务完成
        future.get();
        System.out.println("主程序结束");
    }
}
```

### 3.3 线程池选择策略

根据任务特性选择合适的线程池至关重要：

**CPU密集型任务**：
```java
// 使用ForkJoinPool，并行度等于CPU核心数
private static final ForkJoinPool cpuIntensivePool = new ForkJoinPool(
    Runtime.getRuntime().availableProcessors()
);

CompletableFuture.supplyAsync(() -> {
    // CPU密集型计算
    return intensiveCalculation();
}, cpuIntensivePool);
```

**IO密集型任务**：
```java
// 使用ThreadPoolExecutor，配置更多线程
private static final ThreadPoolExecutor ioIntensivePool = new ThreadPoolExecutor(
    10,  // 核心线程数
    20,  // 最大线程数
    60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100)
);

CompletableFuture.supplyAsync(() -> {
    // IO密集型操作，如网络请求、数据库查询
    return databaseQuery();
}, ioIntensivePool);
```

## 四、链式操作：thenApply与thenAccept

### 4.1 thenApply：结果转换

`thenApply`方法用于对前一个阶段的结果进行转换，返回一个新的`CompletableFuture`。

**基础转换示例**：
```java
public class ThenApplyExample {
    public static void main(String[] args) throws Exception {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            return "Hello";
        }).thenApply(result -> {
            // 对上游结果进行转换
            return result + " World";
        }).thenApply(result -> {
            // 继续转换
            return result.toUpperCase();
        });
        
        System.out.println("最终结果: " + future.get()); // 输出: HELLO WORLD
    }
}
```

**实战案例：数据处理流水线**：
```java
public class DataProcessingPipeline {
    public static void main(String[] args) throws Exception {
        CompletableFuture<String> processingPipeline = CompletableFuture.supplyAsync(() -> {
            // 第一阶段：数据获取
            System.out.println("获取原始数据...");
            return "raw_data_123";
        }).thenApply(rawData -> {
            // 第二阶段：数据清洗
            System.out.println("清洗数据: " + rawData);
            return rawData.replace("raw_", "cleaned_");
        }).thenApply(cleanedData -> {
            // 第三阶段：数据转换
            System.out.println("转换数据: " + cleanedData);
            return cleanedData.toUpperCase();
        }).thenApply(transformedData -> {
            // 第四阶段：数据存储准备
            System.out.println("准备存储: " + transformedData);
            return "final_" + transformedData;
        });
        
        String finalResult = processingPipeline.get();
        System.out.println("处理完成: " + finalResult);
    }
}
```

### 4.2 thenAccept：结果消费

`thenAccept`方法接受一个`Consumer`参数，对结果进行消费但不返回新值。

**基础用法**：
```java
public class ThenAcceptExample {
    public static void main(String[] args) throws Exception {
        CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
            return "处理后的数据";
        }).thenAccept(result -> {
            // 消费结果，如打印、保存等
            System.out.println("消费结果: " + result);
        });
        
        future.get(); // 等待整个流水线完成
    }
}
```

**实战案例：异步处理结果**：
```java
public class AsyncResultHandler {
    public static void main(String[] args) throws Exception {
        CompletableFuture<Void> processingFlow = CompletableFuture.supplyAsync(() -> {
            // 模拟业务数据处理
            System.out.println("开始业务处理...");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new IllegalStateException(e);
            }
            return new BusinessData(1, "处理完成", System.currentTimeMillis());
        }).thenAccept(businessData -> {
            // 异步处理业务结果
            System.out.println("收到业务数据: " + businessData);
            // 可以在这里进行结果保存、通知等操作
            saveToDatabase(businessData);
            sendNotification(businessData);
        });
        
        System.out.println("主线程继续执行其他任务...");
        processingFlow.get(); // 等待异步处理完成
        System.out.println("所有处理完成");
    }
    
    private static void saveToDatabase(BusinessData data) {
        System.out.println("保存到数据库: " + data);
    }
    
    private static void sendNotification(BusinessData data) {
        System.out.println("发送通知: " + data);
    }
    
    static class BusinessData {
        int id;
        String status;
        long timestamp;
        
        BusinessData(int id, String status, long timestamp) {
            this.id = id;
            this.status = status;
            this.timestamp = timestamp;
        }
        
        @Override
        public String toString() {
            return String.format("BusinessData{id=%d, status='%s', timestamp=%d}", 
                               id, status, timestamp);
        }
    }
}
```

### 4.3 同步与异步执行模式

`thenApply`和`thenAccept`都有同步和异步版本：

**同步执行**（默认）：
```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello")
    .thenApply(result -> {  // 在上一个任务的同一线程执行
        return result + " World";
    });
```

**异步执行**（thenApplyAsync）：
```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello")
    .thenApplyAsync(result -> {  // 在默认线程池执行
        return result + " World";
    });
```

**指定线程池的异步执行**：
```java
Executor customExecutor = Executors.newFixedThreadPool(3);
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello")
    .thenApplyAsync(result -> {
        return result + " World";
    }, customExecutor);
```

## 五、实战案例：电商价格计算系统

下面通过一个完整的电商价格计算案例，展示CompletableFuture在实际业务中的应用。

### 5.1 业务场景描述

假设我们需要计算一个商品的最终价格，计算过程涉及：
1. 获取商品基础价格
2. 查询用户会员等级折扣
3. 查询当前促销活动折扣
4. 计算运费
5. 汇总最终价格

### 5.2 传统同步实现 vs CompletableFuture异步实现

**传统同步实现**（性能较差）：
```java
public class SyncPriceCalculator {
    public BigDecimal calculatePrice(Long productId, Long userId) {
        // 1. 获取商品基础价格（耗时100ms）
        BigDecimal basePrice = productService.getBasePrice(productId);
        
        // 2. 查询用户折扣（耗时150ms）
        BigDecimal userDiscount = userService.getUserDiscount(userId);
        
        // 3. 查询促销折扣（耗时200ms）
        BigDecimal promotionDiscount = promotionService.getCurrentDiscount(productId);
        
        // 4. 计算运费（耗时100ms）
        BigDecimal shippingFee = shippingService.calculateFee(productId, userId);
        
        // 5. 计算最终价格
        BigDecimal finalPrice = basePrice
            .multiply(userDiscount)
            .multiply(promotionDiscount)
            .add(shippingFee);
            
        return finalPrice;
    }
}
// 总耗时：100 + 150 + 200 + 100 = 550ms
```

**CompletableFuture异步实现**（性能优化）：
```java
public class AsyncPriceCalculator {
    private final ExecutorService executor = Executors.newFixedThreadPool(4);
    
    public CompletableFuture<BigDecimal> calculatePrice(Long productId, Long userId) {
        // 并行执行所有查询任务
        CompletableFuture<BigDecimal> basePriceFuture = 
            CompletableFuture.supplyAsync(() -> 
                productService.getBasePrice(productId), executor);
                
        CompletableFuture<BigDecimal> userDiscountFuture = 
            CompletableFuture.supplyAsync(() -> 
                userService.getUserDiscount(userId), executor);
                
        CompletableFuture<BigDecimal> promotionDiscountFuture = 
            CompletableFuture.supplyAsync(() -> 
                promotionService.getCurrentDiscount(productId), executor);
                
        CompletableFuture<BigDecimal> shippingFeeFuture = 
            CompletableFuture.supplyAsync(() -> 
                shippingService.calculateFee(productId, userId), executor);
        
        // 组合所有结果
        return basePriceFuture
            .thenCombine(userDiscountFuture, BigDecimal::multiply)
            .thenCombine(promotionDiscountFuture, BigDecimal::multiply)
            .thenCombine(shippingFeeFuture, BigDecimal::add)
            .exceptionally(ex -> {
                System.err.println("价格计算失败: " + ex.getMessage());
                return BigDecimal.ZERO; // 返回默认值
            });
    }
}
// 总耗时：最慢的任务时间 ≈ 200ms
```

### 5.3 完整的电商价格服务

```java
public class ECommercePriceService {
    private final ExecutorService priceCalculatorExecutor;
    private final ProductService productService;
    private final UserService userService;
    private final PromotionService promotionService;
    private final ShippingService shippingService;
    
    public ECommercePriceService() {
        this.priceCalculatorExecutor = Executors.newFixedThreadPool(10);
        // 初始化各服务...
    }
    
    public CompletableFuture<PriceResult> calculateDetailedPrice(Long productId, Long userId) {
        // 并行获取所有必要信息
        CompletableFuture<BigDecimal> basePriceFuture = getBasePrice(productId);
        CompletableFuture<BigDecimal> userDiscountFuture = getUserDiscount(userId);
        CompletableFuture<BigDecimal> promotionDiscountFuture = getPromotionDiscount(productId);
        CompletableFuture<BigDecimal> shippingFeeFuture = getShippingFee(productId, userId);
        
        // 组合计算结果
        return basePriceFuture
            .thenCombine(userDiscountFuture, (price, discount) -> {
                BigDecimal discountedPrice = price.multiply(discount);
                return new PriceStepResult(price, discountedPrice, "用户折扣应用");
            })
            .thenCombine(promotionDiscountFuture, (stepResult, promotionDiscount) -> {
                BigDecimal newPrice = stepResult.getCurrentPrice().multiply(promotionDiscount);
                return new PriceStepResult(stepResult.getBasePrice(), newPrice, "促销折扣应用");
            })
            .thenCombine(shippingFeeFuture, (stepResult, shippingFee) -> {
                BigDecimal finalPrice = stepResult.getCurrentPrice().add(shippingFee);
                return new PriceResult(
                    stepResult.getBasePrice(),
                    finalPrice,
                    shippingFee,
                    "价格计算完成"
                );
            })
            .exceptionally(ex -> {
                // 异常处理
                return new PriceResult(
                    BigDecimal.ZERO, BigDecimal.ZERO, BigDecimal.ZERO,
                    "计算失败: " + ex.getMessage()
                );
            });
    }
    
    private CompletableFuture<BigDecimal> getBasePrice(Long productId) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("查询商品基础价格: " + productId);
            return productService.getBasePrice(productId);
        }, priceCalculatorExecutor);
    }
    
    private CompletableFuture<BigDecimal> getUserDiscount(Long userId) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("查询用户折扣: " + userId);
            return userService.getUserDiscount(userId);
        }, priceCalculatorExecutor);
    }
    
    // 其他服务方法类似...
    
    public void shutdown() {
        priceCalculatorExecutor.shutdown();
    }
}
```

## 六、最佳实践与注意事项

### 6.1 线程池管理策略

**避免使用默认线程池**：
```java
// 不推荐：使用默认commonPool（可能造成线程饥饿）
CompletableFuture.supplyAsync(() -> heavyTask());

// 推荐：使用自定义线程池
private static final ExecutorService customExecutor = 
    Executors.newFixedThreadPool(10);

CompletableFuture.supplyAsync(() -> heavyTask(), customExecutor);
```

**线程池配置建议**：
- **CPU密集型**：线程数 = CPU核心数
- **IO密集型**：线程数 = CPU核心数 × 2 ~ 4
- **使用有界队列**防止内存溢出
- **设置合理的拒绝策略**

### 6.2 异常处理基础

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (Math.random() > 0.5) {
        throw new RuntimeException("模拟异常");
    }
    return "成功结果";
}).exceptionally(ex -> {
    // 异常处理，返回默认值
    System.err.println("处理异常: " + ex.getMessage());
    return "默认结果";
});
```

### 6.3 资源清理与超时控制

```java
public class ResourceAwareFuture {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        
        try {
            CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
                // 模拟耗时任务
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return "任务被中断";
                }
                return "任务完成";
            }, executor);
            
            // 设置超时
            String result = future.get(2, TimeUnit.SECONDS);
            System.out.println("结果: " + result);
            
        } catch (TimeoutException e) {
            System.out.println("任务超时");
        } catch (Exception e) {
            System.out.println("其他异常: " + e.getMessage());
        } finally {
            // 重要：及时关闭线程池
            executor.shutdown();
        }
    }
}
```

## 总结

通过本讲的学习，我们掌握了`CompletableFuture`的基础用法和核心概念。关键要点总结：

1. **从Future到CompletableFuture**：解决了传统Future的局限性，提供了更强大的异步编程能力
2. **任务创建**：使用`supplyAsync`和`runAsync`创建异步任务，注意线程池选择
3. **链式操作**：`thenApply`用于结果转换，`thenAccept`用于结果消费
4. **实战应用**：通过电商价格计算案例展示了异步编程的性能优势
5. **最佳实践**：合理管理线程池，做好异常处理和资源清理

`CompletableFuture`将回调模式转变为声明式组合，让异步编程变得更加直观和优雅。在下一讲中，我们将深入探讨更高级的特性，包括任务组合、异常处理、超时控制等高级用法。

**思考与实践**：在你的项目中，哪些耗时操作可以用`CompletableFuture`进行优化？尝试用本讲知识重构一个同步业务逻辑，体验性能提升的效果！

【转载须知】：**转载请注明原文出处及作者信息**