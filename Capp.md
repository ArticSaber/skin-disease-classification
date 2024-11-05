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
