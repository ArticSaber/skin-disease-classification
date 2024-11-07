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
