Understood. Even with indexing only on the `BatchStatusEntity` and `AnalysisEntity` tables, we can still optimize the processing logic. Here is the complete code for the service, repository, and entity classes, considering the constraints.

### Entities

**BatchStatusEntity.java**
```java
import javax.persistence.*;

@Entity
public class BatchStatusEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String batchName;

    @Enumerated(EnumType.STRING)
    private BatchStatus batchStatus;

    // Getters and Setters
}

enum BatchStatus {
    UNPROCESSED,
    PROCESSED,
    PROCESS_ERROR
}
```

**RawDataEntity.java**
```java
import javax.persistence.*;

@Entity
public class RawDataEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String tradeDate;
    private String symbol;
    private String securityName;
    private String clientName;
    private String buySell;
    private int quantityTraded;
    private double tradePrice;
    private String remarks;
    private String batchName;
    private String recordStatus;

    // Getters and Setters
}
```

**AnalysisEntity.java**
```java
import javax.persistence.*;

@Entity
@IdClass(AnalysisEntityId.class)
@Table(name = "analysis_entity", indexes = {
    @Index(name = "idx_trade_date_symbol", columnList = "tradeDate, symbol")
})
public class AnalysisEntity {
    @Id
    private String tradeDate;

    @Id
    private String symbol;

    private double tradePrice;
    private Double previousEma;

    // Getters and Setters
}

import java.io.Serializable;

public class AnalysisEntityId implements Serializable {
    private String tradeDate;
    private String symbol;

    // Getters, Setters, hashCode, equals
}
```

### Repositories

**BatchStatusRepository.java**
```java
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface BatchStatusRepository extends JpaRepository<BatchStatusEntity, Long> {
    BatchStatusEntity findByBatchName(String batchName);
    List<BatchStatusEntity> findByBatchStatus(BatchStatus batchStatus);
}
```

**RawDataRepository.java**
```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface RawDataRepository extends JpaRepository<RawDataEntity, Long> {
    @Query("SELECT r FROM RawDataEntity r WHERE r.batchName = :batchName AND r.recordStatus = :recordStatus ORDER BY r.id ASC")
    List<RawDataEntity> findByBatchNameAndRecordStatus(@Param("batchName") String batchName, @Param("recordStatus") String recordStatus, @Param("limit") int limit, @Param("offset") int offset);

    List<RawDataEntity> findByRecordStatus(String recordStatus);
}
```

**AnalysisRepository.java**
```java
import org.springframework.data.jpa.repository.JpaRepository;

public interface AnalysisRepository extends JpaRepository<AnalysisEntity, AnalysisEntityId> {
    AnalysisEntity findByTradeDateAndSymbol(String tradeDate, String symbol);
}
```

### Service

**BatchProcessingService.java**
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

@Service
public class BatchProcessingService {

    @Autowired
    private BatchStatusRepository batchStatusRepository;

    @Autowired
    private RawDataRepository rawDataRepository;

    @Autowired
    private AnalysisRepository analysisRepository;

    private final ForkJoinPool forkJoinPool = new ForkJoinPool();

    public void processBatches() {
        List<BatchStatusEntity> unprocessedBatches = batchStatusRepository.findByBatchStatus(BatchStatus.UNPROCESSED);

        if (unprocessedBatches.isEmpty()) {
            List<RawDataEntity> unprocessedRecords = rawDataRepository.findByRecordStatus("UNPROCESSED");
            for (RawDataEntity record : unprocessedRecords) {
                if (batchStatusRepository.findByBatchName(record.getBatchName()) == null) {
                    BatchStatusEntity newBatch = new BatchStatusEntity();
                    newBatch.setBatchName(record.getBatchName());
                    newBatch.setBatchStatus(BatchStatus.UNPROCESSED);
                    batchStatusRepository.save(newBatch);
                    unprocessedBatches.add(newBatch);
                }
            }
        }

        for (BatchStatusEntity batch : unprocessedBatches) {
            forkJoinPool.submit(new BatchTask(batch));
        }
    }

    private class BatchTask extends RecursiveAction {
        private final BatchStatusEntity batch;
        private static final int BATCH_SIZE = 1000;

        public BatchTask(BatchStatusEntity batch) {
            this.batch = batch;
        }

        @Override
        protected void compute() {
            try {
                int offset = 0;
                List<RawDataEntity> records;
                do {
                    records = rawDataRepository.findByBatchNameAndRecordStatus(batch.getBatchName(), "UNPROCESSED", BATCH_SIZE, offset);
                    offset += BATCH_SIZE;
                    processRecords(records);
                } while (!records.isEmpty());

                batch.setBatchStatus(BatchStatus.PROCESSED);
            } catch (Exception e) {
                batch.setBatchStatus(BatchStatus.PROCESS_ERROR);
            } finally {
                batchStatusRepository.save(batch);
            }
        }

        @Transactional
        private void processRecords(List<RawDataEntity> records) {
            for (RawDataEntity record : records) {
                AnalysisEntity analysis = analysisRepository.findByTradeDateAndSymbol(record.getTradeDate(), record.getSymbol());
                if (analysis != null && analysis.getTradePrice() == record.getTradePrice()) {
                    record.setRecordStatus("DUPLICATE");
                } else {
                    analysis = new AnalysisEntity();
                    analysis.setTradeDate(record.getTradeDate());
                    analysis.setSymbol(record.getSymbol());
                    analysis.setTradePrice(record.getTradePrice());
                    analysisRepository.save(analysis);
                    record.setRecordStatus("PROCESSED");
                }
                rawDataRepository.save(record);
            }
        }
    }
}
```

### Controller

**BatchProcessingController.java**
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class BatchProcessingController {

    @Autowired
    private BatchProcessingService batchProcessingService;

    @GetMapping("/process-batches")
    public String processBatches() {
        batchProcessingService.processBatches();
        return "Batch processing started";
    }
}
```

### Indexing

Ensure that the database has indexes on the following columns in the `BatchStatusEntity` and `AnalysisEntity` tables:

**Example SQL Commands for Indexes**

```sql
-- Indexes for BatchStatusEntity table
CREATE INDEX idx_batch_name ON batch_status_entity(batch_name);

-- Indexes for AnalysisEntity table
CREATE INDEX idx_trade_date_symbol ON analysis_entity(trade_date, symbol);
```

By implementing these optimizations, you can ensure efficient and ordered processing of records, even for large datasets. The code will work without indexes, but adding indexes will significantly improve performance, especially for large datasets.


```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.concurrent.Executor;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

@Service
public class BatchProcessingService {

    @Autowired
    private BatchStatusRepository batchStatusRepository;

    @Autowired
    private RawDataRepository rawDataRepository;

    @Autowired
    private AnalysisRepository analysisRepository;

    @Autowired
    private Executor taskExecutor;

    @Autowired
    private ForkJoinPool forkJoinPool;

    public void processBatches() {
        List<BatchStatusEntity> unprocessedBatches = batchStatusRepository.findByBatchStatus(BatchStatus.UNPROCESSED);

        if (unprocessedBatches.isEmpty()) {
            List<RawDataEntity> unprocessedRecords = rawDataRepository.findByRecordStatus("UNPROCESSED");
            for (RawDataEntity record : unprocessedRecords) {
                if (batchStatusRepository.findByBatchName(record.getBatchName()) == null) {
                    BatchStatusEntity newBatch = new BatchStatusEntity();
                    newBatch.setBatchName(record.getBatchName());
                    newBatch.setBatchStatus(BatchStatus.UNPROCESSED);
                    batchStatusRepository.save(newBatch);
                    unprocessedBatches.add(newBatch);
                }
            }
        }

        for (BatchStatusEntity batch : unprocessedBatches) {
            taskExecutor.execute(new BatchTask(batch));
        }
    }

    private class BatchTask implements Runnable {
        private final BatchStatusEntity batch;
        private static final int BATCH_SIZE = 1000;

        public BatchTask(BatchStatusEntity batch) {
            this.batch = batch;
        }

        @Override
        public void run() {
            try {
                int offset = 0;
                List<RawDataEntity> records;
                do {
                    records = rawDataRepository.findByBatchNameAndRecordStatus(batch.getBatchName(), "UNPROCESSED", BATCH_SIZE, offset);
                    offset += BATCH_SIZE;
                    forkJoinPool.invoke(new RecordProcessingTask(records));
                } while (!records.isEmpty());

                batch.setBatchStatus(BatchStatus.PROCESSED);
            } catch (Exception e) {
                batch.setBatchStatus(BatchStatus.PROCESS_ERROR);
            } finally {
                batchStatusRepository.save(batch);
            }
        }
    }

    private class RecordProcessingTask extends RecursiveAction {
        private final List<RawDataEntity> records;

        public RecordProcessingTask(List<RawDataEntity> records) {
            this.records = records;
        }

        @Override
        protected void compute() {
            for (RawDataEntity record : records) {
                AnalysisEntity analysis = analysisRepository.findByTradeDateAndSymbol(record.getTradeDate(), record.getSymbol());
                if (analysis != null && analysis.getTradePrice() == record.getTradePrice()) {
                    record.setRecordStatus("DUPLICATE");
                } else {
                    analysis = new AnalysisEntity();
                    analysis.setTradeDate(record.getTradeDate());
                    analysis.setSymbol(record.getSymbol());
                    analysis.setTradePrice(record.getTradePrice());
                    analysisRepository.save(analysis);
                    record.setRecordStatus("PROCESSED");
                }
                rawDataRepository.save(record);
            }
        }
    }
}
```
```
public void processBatches() {
    List<BatchStatusEntity> unprocessedBatches = batchStatusRepository.findByBatchStatus(BatchStatus.UNPROCESSED);

    if (unprocessedBatches.isEmpty()) {
        // Retrieve distinct batch names with "UNPROCESSED" status
        List<String> unprocessedBatchNames = rawDataRepository.findDistinctBatchNamesByRecordStatus("UNPROCESSED");
        
        for (String batchName : unprocessedBatchNames) {
            // Check if the batch already exists in BatchStatus table
            if (batchStatusRepository.findByBatchName(batchName) == null) {
                // Create a new BatchStatusEntity if it does not exist
                BatchStatusEntity newBatch = new BatchStatusEntity();
                newBatch.setBatchName(batchName);
                newBatch.setBatchStatus(BatchStatus.UNPROCESSED);
                
                // Add it to the unprocessed batch list
                unprocessedBatches.add(newBatch);
            }
        }
        
        // Save all new batch entries in a single batch insert to optimize performance
        batchStatusRepository.saveAll(unprocessedBatches);
    }

    // Process each batch asynchronously
    for (BatchStatusEntity batch : unprocessedBatches) {
        taskExecutor.execute(new BatchTask(batch));
    }
}
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface RawDataRepository extends JpaRepository<RawDataEntity, Long> {
    
    @Query("SELECT DISTINCT r.batchName FROM RawDataEntity r WHERE r.recordStatus = :status")
    List<String> findDistinctBatchNamesByRecordStatus(@Param("status") String status);

    List<RawDataEntity> findByRecordStatus(String recordStatus);
}
```





```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.List;
import java.util.concurrent.Executor;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;
import java.util.concurrent.atomic.AtomicInteger;

@Service
public class BatchProcessingService {

    @Autowired
    private BatchStatusRepository batchStatusRepository;

    @Autowired
    private RawDataRepository rawDataRepository;

    @Autowired
    private AnalysisRepository analysisRepository;

    @Autowired
    private ThreadPoolTaskExecutor taskExecutor;

    @Autowired
    private ForkJoinPool forkJoinPool;

    private AtomicInteger processedCount = new AtomicInteger(0);

    @Async("taskExecutor")
    public void processBatches() {
        List<BatchStatusEntity> unprocessedBatches = batchStatusRepository.findByBatchStatus(BatchStatus.UNPROCESSED);

        if (unprocessedBatches.isEmpty()) {
            List<RawDataEntity> unprocessedRecords = rawDataRepository.findByRecordStatus("UNPROCESSED");
            for (RawDataEntity record : unprocessedRecords) {
                if (batchStatusRepository.findByBatchName(record.getBatchName()) == null) {
                    BatchStatusEntity newBatch = new BatchStatusEntity();
                    newBatch.setBatchName(record.getBatchName());
                    newBatch.setBatchStatus(BatchStatus.UNPROCESSED);
                    batchStatusRepository.save(newBatch);
                    unprocessedBatches.add(newBatch);
                }
            }
        }

        for (BatchStatusEntity batch : unprocessedBatches) {
            taskExecutor.execute(() -> processBatch(batch));
        }
    }

    @Transactional
    public void processBatch(BatchStatusEntity batch) {
        try {
            int pageNumber = 0;
            Page<RawDataEntity> page;
            do {
                Pageable pageable = PageRequest.of(pageNumber, BATCH_SIZE);
                page = rawDataRepository.findByBatchNameAndRecordStatus(batch.getBatchName(), "UNPROCESSED", pageable);
                forkJoinPool.invoke(new RecordProcessingTask(page.getContent()));
                pageNumber++;
            } while (page.hasNext());

            batch.setBatchStatus(BatchStatus.PROCESSED);
        } catch (Exception e) {
            batch.setBatchStatus(BatchStatus.PROCESS_ERROR);
        } finally {
            batchStatusRepository.saveAndFlush(batch);
        }
    }

    private class RecordProcessingTask extends RecursiveAction {
        private final List<RawDataEntity> records;

        public RecordProcessingTask(List<RawDataEntity> records) {
            this.records = records;
        }

        @Override
        @Transactional
        protected void compute() {
            for (RawDataEntity record : records) {
                try {
                    AnalysisEntityId analysisId = new AnalysisEntityId(record.getTradeDate(), record.getSymbol());
                    AnalysisEntity analysis = analysisRepository.findById(analysisId).orElse(null);
                    if (analysis != null && analysis.getTradePrice() == record.getTradePrice()) {
                        record.setRecordStatus("DUPLICATE");
                    } else {
                        analysis = new AnalysisEntity();
                        analysis.setId(analysisId);
                        analysis.setTradePrice(record.getTradePrice());
                        analysisRepository.save(analysis);
                        record.setRecordStatus("PROCESSED");
                    }
                    rawDataRepository.saveAndFlush(record);
                    int count = processedCount.incrementAndGet();
                    System.out.println("Processed record number: " + count);
                } catch (Exception e) {
                    record.setRecordStatus("PROCESS_ERROR");
                    rawDataRepository.saveAndFlush(record);
                }
            }
        }
    }

    public int getProcessedCount() {
        return processedCount.get();
    }
}
```


##claude

```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;

@Service
public class BatchProcessingService {
    private static final int THRESHOLD = 100;
    private static final int BATCH_SIZE = 1000;

    @Autowired
    private BatchStatusRepository batchStatusRepository;

    @Autowired
    private RawDataRepository rawDataRepository;

    @Autowired
    private AnalysisRepository analysisRepository;

    @Autowired
    private ThreadPoolTaskExecutor taskExecutor;

    @Autowired
    private ForkJoinPool forkJoinPool;

    private final AtomicInteger processedCount = new AtomicInteger(0);
    private final AtomicInteger errorCount = new AtomicInteger(0);

    @Async("taskExecutor")
    public CompletableFuture<ProcessingSummary> processBatches() {
        List<BatchStatusEntity> unprocessedBatches = createOrGetUnprocessedBatches();
        
        List<CompletableFuture<BatchResult>> futures = unprocessedBatches.stream()
            .map(batch -> CompletableFuture.supplyAsync(
                () -> processBatch(batch), 
                taskExecutor
            ))
            .collect(Collectors.toList());

        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> summarizeResults(futures));
    }

    private List<BatchStatusEntity> createOrGetUnprocessedBatches() {
        List<BatchStatusEntity> unprocessedBatches = 
            batchStatusRepository.findByBatchStatus(BatchStatus.UNPROCESSED);

        if (unprocessedBatches.isEmpty()) {
            Map<String, List<RawDataEntity>> recordsByBatch = 
                rawDataRepository.findByRecordStatus("UNPROCESSED")
                    .parallelStream()
                    .collect(Collectors.groupingBy(RawDataEntity::getBatchName));

            List<BatchStatusEntity> newBatches = recordsByBatch.keySet().stream()
                .filter(batchName -> batchStatusRepository.findByBatchName(batchName) == null)
                .map(batchName -> {
                    BatchStatusEntity batch = new BatchStatusEntity();
                    batch.setBatchName(batchName);
                    batch.setBatchStatus(BatchStatus.UNPROCESSED);
                    return batch;
                })
                .collect(Collectors.toList());

            if (!newBatches.isEmpty()) {
                batchStatusRepository.saveAll(newBatches);
                unprocessedBatches.addAll(newBatches);
            }
        }
        return unprocessedBatches;
    }

    @Transactional
    public BatchResult processBatch(BatchStatusEntity batch) {
        BatchResult result = new BatchResult(batch.getBatchName());
        try {
            int pageNumber = 0;
            Page<RawDataEntity> page;
            do {
                Pageable pageable = PageRequest.of(pageNumber, BATCH_SIZE);
                page = rawDataRepository.findByBatchNameAndRecordStatus(
                    batch.getBatchName(), 
                    "UNPROCESSED", 
                    pageable
                );
                
                RecordProcessingTask task = new RecordProcessingTask(page.getContent());
                forkJoinPool.invoke(task);
                
                result.addProcessed(task.getProcessedCount());
                result.addErrors(task.getErrorCount());
                
                pageNumber++;
            } while (page.hasNext());

            batch.setBatchStatus(BatchStatus.PROCESSED);
            batch.setProcessedCount(result.getProcessedCount());
            batch.setErrorCount(result.getErrorCount());
            
        } catch (Exception e) {
            batch.setBatchStatus(BatchStatus.PROCESS_ERROR);
            result.setFailed(true);
            result.setError(e.getMessage());
        } finally {
            batchStatusRepository.saveAndFlush(batch);
        }
        return result;
    }

    private class RecordProcessingTask extends RecursiveAction {
        private final List<RawDataEntity> records;
        private int processedCount = 0;
        private int errorCount = 0;

        public RecordProcessingTask(List<RawDataEntity> records) {
            this.records = records;
        }

        @Override
        protected void compute() {
            if (records.size() <= THRESHOLD) {
                processRecordBatch();
            } else {
                int mid = records.size() / 2;
                RecordProcessingTask left = new RecordProcessingTask(
                    records.subList(0, mid)
                );
                RecordProcessingTask right = new RecordProcessingTask(
                    records.subList(mid, records.size())
                );
                invokeAll(left, right);
                
                processedCount = left.getProcessedCount() + right.getProcessedCount();
                errorCount = left.getErrorCount() + right.getErrorCount();
            }
        }

        @Transactional
        private void processRecordBatch() {
            List<AnalysisEntity> analysesToSave = new ArrayList<>();
            List<RawDataEntity> recordsToUpdate = new ArrayList<>();

            for (RawDataEntity record : records) {
                try {
                    processRecord(record, analysesToSave, recordsToUpdate);
                    processedCount++;
                } catch (Exception e) {
                    handleRecordError(record, recordsToUpdate);
                    errorCount++;
                }
            }

            saveProcessedRecords(analysesToSave, recordsToUpdate);
        }

        private void processRecord(
            RawDataEntity record, 
            List<AnalysisEntity> analysesToSave,
            List<RawDataEntity> recordsToUpdate
        ) {
            AnalysisEntityId analysisId = new AnalysisEntityId(
                record.getTradeDate(), 
                record.getSymbol()
            );
            
            AnalysisEntity analysis = analysisRepository
                .findById(analysisId)
                .orElse(null);

            if (analysis != null && analysis.getTradePrice() == record.getTradePrice()) {
                record.setRecordStatus("DUPLICATE");
            } else {
                analysis = new AnalysisEntity();
                analysis.setId(analysisId);
                analysis.setTradePrice(record.getTradePrice());
                analysesToSave.add(analysis);
                record.setRecordStatus("PROCESSED");
            }
            recordsToUpdate.add(record);
        }

        private void handleRecordError(
            RawDataEntity record,
            List<RawDataEntity> recordsToUpdate
        ) {
            record.setRecordStatus("PROCESS_ERROR");
            recordsToUpdate.add(record);
        }

        @Transactional
        private void saveProcessedRecords(
            List<AnalysisEntity> analysesToSave,
            List<RawDataEntity> recordsToUpdate
        ) {
            if (!analysesToSave.isEmpty()) {
                analysisRepository.saveAll(analysesToSave);
            }
            if (!recordsToUpdate.isEmpty()) {
                rawDataRepository.saveAll(recordsToUpdate);
            }
        }

        public int getProcessedCount() {
            return processedCount;
        }

        public int getErrorCount() {
            return errorCount;
        }
    }

    @Getter
    @AllArgsConstructor
    public static class BatchResult {
        private final String batchName;
        private int processedCount = 0;
        private int errorCount = 0;
        private boolean failed = false;
        private String error;

        public void addProcessed(int count) {
            this.processedCount += count;
        }

        public void addErrors(int count) {
            this.errorCount += count;
        }

        public void setFailed(boolean failed) {
            this.failed = failed;
        }

        public void setError(String error) {
            this.error = error;
        }
    }

    @Getter
    public static class ProcessingSummary {
        private final int totalProcessed;
        private final int totalErrors;
        private final List<BatchResult> batchResults;

        public ProcessingSummary(List<BatchResult> results) {
            this.batchResults = results;
            this.totalProcessed = results.stream()
                .mapToInt(BatchResult::getProcessedCount)
                .sum();
            this.totalErrors = results.stream()
                .mapToInt(BatchResult::getErrorCount)
                .sum();
        }
    }

    private ProcessingSummary summarizeResults(
        List<CompletableFuture<BatchResult>> futures
    ) {
        List<BatchResult> results = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        return new ProcessingSummary(results);
    }

    public int getProcessedCount() {
        return processedCount.get();
    }

    public int getErrorCount() {
        return errorCount.get();
    }
}
```
```
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Value("${thread.pool.core-size:10}")
    private int corePoolSize;

    @Value("${thread.pool.max-size:50}")
    private int maxPoolSize;

    @Value("${thread.pool.queue-capacity:500}")
    private int queueCapacity;

    @Value("${fork.join.parallelism:4}")
    private int forkJoinParallelism;

    @Bean(name = "taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(maxPoolSize);
        executor.setQueueCapacity(queueCapacity);
        executor.setThreadNamePrefix("batch-processor-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.initialize();
        return executor;
    }

    @Bean
    public ForkJoinPool forkJoinPool() {
        return new ForkJoinPool(forkJoinParallelism);
    }

    @Override
    public Executor getAsyncExecutor() {
        return taskExecutor();
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new SimpleAsyncUncaughtExceptionHandler();
    }
}
```
```
# application.properties
thread.pool.core-size=10
thread.pool.max-size=50
thread.pool.queue-capacity=500
fork.join.parallelism=4
```

##type 2

```
# application.properties
thread.pool.core-size=10
thread.pool.max-size=50
thread.pool.queue-capacity=500
fork.join.parallelism=4
```
```
// BatchProcessingService.java
@Service
@Slf4j
@RequiredArgsConstructor
public class BatchProcessingService {
    private static final int BATCH_SIZE = 5000;
    private static final int CHUNK_SIZE = 200;

    private final BatchStatusRepository batchStatusRepository;
    private final RawDataRepository rawDataRepository;
    private final AnalysisRepository analysisRepository;
    private final ThreadPoolTaskExecutor taskExecutor;
    private final ForkJoinPool forkJoinPool;
    private final JdbcTemplate jdbcTemplate;

    private final AtomicInteger processedCount = new AtomicInteger(0);
    private final AtomicInteger errorCount = new AtomicInteger(0);

    @Async("taskExecutor")
    public CompletableFuture<ProcessingSummary> processBatches() {
        StopWatch watch = new StopWatch();
        watch.start();
        log.info("Starting batch processing");

        try {
            List<BatchStatusEntity> batches = createOrGetUnprocessedBatches();
            List<CompletableFuture<BatchResult>> futures = batches.stream()
                .map(batch -> CompletableFuture.supplyAsync(
                    () -> processBatch(batch), 
                    taskExecutor
                ))
                .collect(Collectors.toList());

            return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                .thenApply(v -> {
                    watch.stop();
                    ProcessingSummary summary = summarizeResults(futures);
                    log.info("Batch processing completed in {} ms", watch.getTotalTimeMillis());
                    return summary;
                });
        } catch (Exception e) {
            log.error("Batch processing failed", e);
            throw new BatchProcessingException("Failed to process batches", e);
        }
    }

    @Transactional(readOnly = true)
    protected List<BatchStatusEntity> createOrGetUnprocessedBatches() {
        List<BatchStatusEntity> unprocessedBatches = 
            batchStatusRepository.findByBatchStatus(BatchStatus.UNPROCESSED);

        if (unprocessedBatches.isEmpty()) {
            try (Stream<RawDataEntity> stream = rawDataRepository.streamUnprocessedRecords()) {
                Map<String, List<RawDataEntity>> recordsByBatch = stream
                    .parallel()
                    .collect(Collectors.groupingByConcurrent(RawDataEntity::getBatchName));

                List<BatchStatusEntity> newBatches = recordsByBatch.keySet().stream()
                    .map(this::createNewBatch)
                    .collect(Collectors.toList());

                return batchStatusRepository.saveAll(newBatches);
            }
        }
        return unprocessedBatches;
    }

    @Transactional
    protected BatchResult processBatch(BatchStatusEntity batch) {
        BatchResult result = new BatchResult(batch.getBatchName());
        StopWatch watch = new StopWatch();
        watch.start();

        try {
            try (Stream<RawDataEntity> recordStream = 
                rawDataRepository.streamByBatchName(batch.getBatchName())) {
                
                List<List<RawDataEntity>> chunks = StreamSupport
                    .stream(Iterables.partition(recordStream::iterator, CHUNK_SIZE)
                    .spliterator(), false)
                    .collect(Collectors.toList());

                forkJoinPool.submit(() -> 
                    chunks.parallelStream().forEach(chunk -> 
                        processChunk(chunk, result)))
                    .get(30, TimeUnit.MINUTES);

                batch.setBatchStatus(BatchStatus.PROCESSED);
                batch.setProcessedCount(result.getProcessedCount());
                batch.setErrorCount(result.getErrorCount());
            }
        } catch (Exception e) {
            log.error("Failed to process batch: {}", batch.getBatchName(), e);
            batch.setBatchStatus(BatchStatus.PROCESS_ERROR);
            result.setFailed(true);
            result.setError(e.getMessage());
        } finally {
            watch.stop();
            log.info("Batch {} processed in {} ms", batch.getBatchName(), 
                watch.getTotalTimeMillis());
            batchStatusRepository.save(batch);
        }
        return result;
    }

    @Transactional
    protected void processChunk(List<RawDataEntity> records, BatchResult result) {
        List<AnalysisEntity> analysesToSave = new ArrayList<>();
        List<RawDataEntity> recordsToUpdate = new ArrayList<>();

        for (RawDataEntity record : records) {
            try {
                processRecord(record, analysesToSave, recordsToUpdate);
                result.incrementProcessed();
                processedCount.incrementAndGet();
            } catch (Exception e) {
                log.error("Error processing record: {}", record.getId(), e);
                record.setRecordStatus("PROCESS_ERROR");
                recordsToUpdate.add(record);
                result.incrementErrors();
                errorCount.incrementAndGet();
            }
        }

        executeBatchUpdates(analysesToSave, recordsToUpdate);
    }

    private void executeBatchUpdates(
        List<AnalysisEntity> analysesToSave, 
        List<RawDataEntity> recordsToUpdate
    ) {
        if (!analysesToSave.isEmpty()) {
            jdbcTemplate.batchUpdate(
                "INSERT INTO analysis (trade_date, symbol, trade_price) VALUES (?,?,?)",
                new BatchPreparedStatementSetter() {
                    @Override
                    public void setValues(PreparedStatement ps, int i) throws SQLException {
                        AnalysisEntity analysis = analysesToSave.get(i);
                        ps.setDate(1, analysis.getTradeDate());
                        ps.setString(2, analysis.getSymbol());
                        ps.setBigDecimal(3, analysis.getTradePrice());
                    }

                    @Override
                    public int getBatchSize() {
                        return analysesToSave.size();
                    }
                }
            );
        }

        if (!recordsToUpdate.isEmpty()) {
            jdbcTemplate.batchUpdate(
                "UPDATE raw_data SET status = ? WHERE id = ?",
                new BatchPreparedStatementSetter() {
                    @Override
                    public void setValues(PreparedStatement ps, int i) throws SQLException {
                        RawDataEntity record = recordsToUpdate.get(i);
                        ps.setString(1, record.getRecordStatus());
                        ps.setLong(2, record.getId());
                    }

                    @Override
                    public int getBatchSize() {
                        return recordsToUpdate.size();
                    }
                }
            );
        }
    }
}
```
```
// AsyncConfig.java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Value("${thread.pool.core-size:20}")
    private int corePoolSize;
    @Value("${thread.pool.max-size:100}")
    private int maxPoolSize;
    @Value("${thread.pool.queue-capacity:1000}")
    private int queueCapacity;
    @Value("${fork.join.parallelism:8}")
    private int forkJoinParallelism;

    @Bean(name = "taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(maxPoolSize);
        executor.setQueueCapacity(queueCapacity);
        executor.setThreadNamePrefix("batch-processor-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.initialize();
        return executor;
    }

    @Bean
    public ForkJoinPool forkJoinPool() {
        return new ForkJoinPool(forkJoinParallelism);
    }

    @Override
    public Executor getAsyncExecutor() {
        return taskExecutor();
    }
}
```
