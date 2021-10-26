---

title: xxl-job搭配redis实现延时队列
date: 2020-08-18 09:55:30
categories:
- 定时任务
tags:
- xxl-job
---

#### 延时任务使用场景

- 保养相关的在指定日期后定时提醒
- 租赁相关的在定标截止时间变更状态

#### 使用xxl-job可以带来哪些好处

1. 可以统一监控任务状态
2. 可以动态调整扫描频率

#### 使用Redis与RDS管理任务的优缺点比较

- Redis
  - 优点
    1. 可以支持更高的扫描频率
    2. 可以zset结构对时间戳进行排序获取更高的查找效率
  - 缺点
    1. 数据稳定性不及RDS，存储大量任务时有丢失风险
- RDS
  - 优点
    1. 数据稳定性更高，丢失风险很低
  - 缺点
    1. 过快的扫描频率会对RDS性能造成影响

#### 代码实现

#### 纯Redis

```java
private static Logger logger = LoggerFactory.getLogger(DelayTask.class);
@Resource
private RedissonClient redissonClient;
@Resource
private ThreadPoolExecutor threadPoolExecutor;

private final Lock lock = new ReentrantLock();

@XxlJob("sendMessageTask")
public ReturnT<String> sendMessageHandler(String param) {
    RScoredSortedSet<ReviewMessage> scoredSortedSet = redissonClient.getScoredSortedSet("delayTask");
    for (int i = 0; i < 100; i++) {
        int machineId = THREAD_LOCAL_RANDOM.nextInt(1000);
        double score = System.currentTimeMillis() + THREAD_LOCAL_RANDOM.nextInt(30000);
        scoredSortedSet.add(score, new ReviewMessage(machineId, "delete machine " + machineId));
    }
    return ReturnT.SUCCESS;
}

@XxlJob("delayTask")
public ReturnT<String> delayTaskJobHandler(String param) throws Exception {
	RScoredSortedSet<ReviewMessage> scoredSortedSet = redissonClient.getScoredSortedSet("delayTask");
    double endScore = (double) System.currentTimeMillis();
    lock.lock();
    try {
    	scoredSortedSet.entryRange(0.0, true, endScore, true).forEach(message -> {
        	threadPoolExecutor.submit(() -> {
            	logger.info("run {} {}", message.getScore(), message.getValue());

			});
        });
        scoredSortedSet.removeRangeByScore(0.0, true, endScore, true);
    } finally {
        lock.unlock();
    }
    return ReturnT.SUCCESS;
}
```

#### 结合RDS防止长期任务数据丢失

```java
private static Logger logger = LoggerFactory.getLogger(DelayTask.class);
@Resource
private RedissonClient redissonClient;
@Resource
private ThreadPoolExecutor threadPoolExecutor;
@Resource
private MessageRecordRepository messageRecordRepository;

private static final ThreadLocalRandom THREAD_LOCAL_RANDOM = ThreadLocalRandom.current();

@PostConstruct
public void init() {
    RScoredSortedSet<ReviewMessage> scoredSortedSet = redissonClient.getScoredSortedSet("delayTask");
    int machineId = THREAD_LOCAL_RANDOM.nextInt(1000);
    double score = System.currentTimeMillis() + THREAD_LOCAL_RANDOM.nextInt(30000);
    ReviewMessage reviewMessage = new ReviewMessage(machineId, "delete machine " + machineId);
    scoredSortedSet.add(score, reviewMessage);
    MessageRecord messageRecord = MessageRecord.builder()
        .type(MessageTypeConstants.REVIEW_MESSAGE)
        .content(GsonTool.toJson(reviewMessage))
        .triggerMillis(score)
        .build();
    messageRecordRepository.save(messageRecord);
}

@XxlJob("sendMessageTask")
public ReturnT<String> sendMessageHandler(String param) {
    RScoredSortedSet<ReviewMessage> scoredSortedSet = redissonClient.getScoredSortedSet("delayTask");
    for (int i = 0; i < 10; i++) {
        int machineId = THREAD_LOCAL_RANDOM.nextInt(1000);
        double score = System.currentTimeMillis() + THREAD_LOCAL_RANDOM.nextInt(30000);
        ReviewMessage reviewMessage = new ReviewMessage(machineId, "delete machine " + machineId);
        MessageRecord messageRecord = MessageRecord.builder()
            .type(MessageTypeConstants.REVIEW_MESSAGE)
            .content(GsonTool.toJson(reviewMessage))
            .triggerMillis(score)
            .build();
        messageRecordRepository.save(messageRecord);
        scoredSortedSet.add(score, reviewMessage);
    }
    return ReturnT.SUCCESS;
}

@XxlJob("delayTask")
public ReturnT<String> delayTaskJobHandler(String param) throws Exception {
    RScoredSortedSet<ReviewMessage> scoredSortedSet = redissonClient.getScoredSortedSet("delayTask");
    threadPoolExecutor.submit(() -> {
        final double endScore = (double) System.currentTimeMillis();
        List<MessageRecord> messageRecords = messageRecordRepository.findByTriggerMillisBetweenAndTypeOrderByTriggerMillisAsc(0.0, endScore, MessageTypeConstants.REVIEW_MESSAGE);
        Collection<ScoredEntry<ReviewMessage>> reviewMessages = scoredSortedSet.entryRange(0.0, true, endScore, true);
        if (messageRecords.size() > reviewMessages.size()) {
            final Set<Double> doubleStream = reviewMessages.stream().map(ScoredEntry::getScore).collect(Collectors.toSet());
            messageRecords.stream().filter(messageRecord -> !doubleStream.contains(messageRecord.getTriggerMillis())).forEach(messageRecord -> {
                ReviewMessage reviewMessage = GsonTool.fromJson(messageRecord.getContent(), ReviewMessage.class);
                logger.info("handle {} {}", messageRecord.getTriggerMillis(), reviewMessage);
            });
        }
        reviewMessages.forEach(reviewMessageScoredEntry -> {
            logger.info("handle {} {}", reviewMessageScoredEntry.getScore(), reviewMessageScoredEntry.getValue());
        });
        scoredSortedSet.removeRangeByScore(0.0, true, endScore, true);
        messageRecordRepository.deleteInBatch(messageRecords);
    });
    
    return ReturnT.SUCCESS;
}
```

