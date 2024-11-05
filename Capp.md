```
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import javax.transaction.Transactional;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;
import java.util.concurrent.ForkJoinPool;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class ProcessingService {

    private final RecordService recordService;
    private final BulkService bulkService;
    private final TestingService testingService;

    @Autowired
    private Executor taskExecutor; // Inject the Executor for async processing

    @Transactional
    public void processRecord() {
        Optional<BulkEntity> optionalBulk = bulkService.getTopNotCompletedBulk();

        if (optionalBulk.isPresent()) {
            BulkEntity bulk = optionalBulk.get();
            processBulkConcurrently(bulk); // Use multithreading for bulk processing
        } else {
            Optional<RecordEntity> optionalRecord = recordService.getFirstMissingBulkRecord(bulkService.getAllBulkNames());

            if (optionalRecord.isPresent()) {
                RecordEntity record = optionalRecord.get();
                String bulkName = record.getBulkname();

                Optional<BulkEntity> existingBulk = bulkService.getBulkByName(bulkName);

                if (existingBulk.isPresent()) {
                    processBulkConcurrently(existingBulk.get());
                } else {
                    BulkEntity newBulk = bulkService.addMissingBulk(bulkName);
                    processBulkConcurrently(newBulk);
                }
            } else {
                throw new RuntimeException("No unprocessed records found in the system.");
            }
        }
    }

    private void processBulkConcurrently(BulkEntity bulk) {
        List<RecordEntity> records = recordService.getAllNotCompletedRecords(bulk.getBulkname());

        // ForkJoinPool for parallel processing of records in bulk
        ForkJoinPool customThreadPool = new ForkJoinPool(Math.min(records.size(), 10)); // Limit to 10 threads or total record count
        try {
            customThreadPool.submit(() -> 
                records.parallelStream()
                    .map(record -> CompletableFuture.runAsync(() -> processRecord(record), taskExecutor))
                    .collect(Collectors.toList())
                    .forEach(CompletableFuture::join)
            ).get();
        } catch (Exception e) {
            throw new RuntimeException("Error in processing bulk records concurrently", e);
        } finally {
            customThreadPool.shutdown();
        }

        // Mark bulk as completed if all records are processed
        if (recordService.areAllRecordsCompletedForBulk(bulk.getBulkname())) {
            bulkService.markBulkAsCompleted(bulk.getBulkname());
        }
    }

    private void processRecord(RecordEntity record) {
        testingService.saveRecordName(record.getRecordname());
        recordService.markRecordAsCompleted(record);
    }

    @Async("taskExecutor")
    public void asyncProcessRecord() {
        processRecord();
    }
}
```







_₹>#&#&÷*×()×?+?!:@>#;;#&₹(÷;÷[×738>4&4[=?=,=(=;₹&₹*₹,,₹&₹,₹₹&&₹&=<<=&=*÷(×[÷;÷&÷*÷*÷
```
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import javax.transaction.Transactional;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;
import java.util.concurrent.ForkJoinPool;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class ProcessingService {

    private final RecordService recordService;
    private final BulkService bulkService;
    private final TestingService testingService;

    @Autowired
    private Executor taskExecutor; // Injected taskExecutor for async processing

    @Transactional
    @Async("taskExecutor")
    public void processBulkAsync() {
        // Step 1: Fetch the top unprocessed bulk and initiate processing
        Optional<BulkEntity> optionalBulk = bulkService.getTopNotCompletedBulk();
        if (optionalBulk.isPresent()) {
            BulkEntity bulk = optionalBulk.get();
            processBulkRecordsConcurrently(bulk); // Process the bulk using ForkJoinPool
        } else {
            // Handle case where no bulks are available
            throw new RuntimeException("No unprocessed bulks found.");
        }
    }

    private void processBulkRecordsConcurrently(BulkEntity bulk) {
        // Step 2: Fetch all NOT_COMPLETED records for this bulk
        List<RecordEntity> records = recordService.getAllNotCompletedRecords(bulk.getBulkname());

        // Use ForkJoinPool for concurrent record processing within the bulk
        ForkJoinPool forkJoinPool = new ForkJoinPool(Math.min(records.size(), 10)); // Set concurrency limit
        try {
            forkJoinPool.submit(() -> 
                records.parallelStream()
                    .map(record -> CompletableFuture.runAsync(() -> processRecord(record), taskExecutor)) // Use CompletableFuture with taskExecutor for each record
                    .collect(Collectors.toList())
                    .forEach(CompletableFuture::join) // Wait for all records in the bulk to finish processing
            ).get();
        } catch (Exception e) {
            throw new RuntimeException("Error processing bulk records concurrently", e);
        } finally {
            forkJoinPool.shutdown(); // Ensure ForkJoinPool is shut down after processing
        }

        // Step 4: Mark bulk as completed if all records are processed
        if (recordService.areAllRecordsCompletedForBulk(bulk.getBulkname())) {
            bulkService.markBulkAsCompleted(bulk.getBulkname());
        }
    }

    private void processRecord(RecordEntity record) {
        // Print thread ID and record name for debugging
        long threadId = Thread.currentThread().getId();
        System.out.println("Processing record: " + record.getRecordname() + " on Thread ID: " + threadId);

        // Save the record name in the testing table and mark it as completed
        testingService.saveRecordName(record.getRecordname());
        recordService.markRecordAsCompleted(record);
    }
}
```
