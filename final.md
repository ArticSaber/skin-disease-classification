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
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;

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

    private static final int BATCH_SIZE = 1000; // Adjust for optimal performance based on memory capacity

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

                // Create a map for deduplication
                Map<AnalysisEntityId, RawDataEntity> deduplicationMap = new ConcurrentHashMap<>();
                List<RawDataEntity> records = page.getContent();

                // Deduplicate and collect processed and duplicate records
                List<RawDataEntity> processedRecords = records.parallelStream()
                    .map(record -> {
                        AnalysisEntityId analysisId = new AnalysisEntityId(record.getTradeDate(), record.getSymbol());
                        RawDataEntity existingRecord = deduplicationMap.putIfAbsent(analysisId, record);

                        if (existingRecord == null) {
                            // Record is unique
                            AnalysisEntity analysis = analysisRepository.findById(analysisId).orElse(null);
                            if (analysis != null && analysis.getTradePrice() == record.getTradePrice()) {
                                record.setRecordStatus("DUPLICATE");
                            } else {
                                // Create or update the analysis entry
                                AnalysisEntity newAnalysis = new AnalysisEntity();
                                newAnalysis.setId(analysisId);
                                newAnalysis.setTradePrice(record.getTradePrice());
                                analysisRepository.save(newAnalysis);
                                record.setRecordStatus("PROCESSED");
                            }
                        } else {
                            // Mark as duplicate if already present in deduplicationMap
                            record.setRecordStatus("DUPLICATE");
                        }
                        return record;
                    })
                    .collect(Collectors.toList());

                // Batch update the statuses for improved performance
                rawDataRepository.saveAll(processedRecords);
                processedCount.addAndGet(processedRecords.size());

                pageNumber++;
            } while (page.hasNext());

            // Update the batch status after all records in the batch are processed
            batch.setBatchStatus(BatchStatus.PROCESSED);
        } catch (Exception e) {
            batch.setBatchStatus(BatchStatus.PROCESS_ERROR);
        } finally {
            batchStatusRepository.saveAndFlush(batch);
        }
    }

    public int getProcessedCount() {
        return processedCount.get();
    }
}
```
