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
                Pageable pageable = PageRequest.of(pageNumber, 1000); // Process in pages of 1000
                page = rawDataRepository.findByBatchNameAndRecordStatus(batch.getBatchName(), "UNPROCESSED", pageable);

                // Parallel process records in the page
                forkJoinPool.submit(() -> processPageInParallel(page.getContent())).get();

                pageNumber++;
            } while (page.hasNext());

            batch.setBatchStatus(BatchStatus.PROCESSED);
        } catch (Exception e) {
            batch.setBatchStatus(BatchStatus.PROCESS_ERROR);
        } finally {
            batchStatusRepository.saveAndFlush(batch);
        }
    }

    private void processPageInParallel(List<RawDataEntity> records) {
        // Use ForkJoinPool to process records in parallel
        ForkJoinTask<Void> task = new ForkJoinTask<Void>() {
            @Override
            public Void fork() {
                records.parallelStream().forEach(record -> processRecord(record));
                return null;
            }

            @Override
            public Void join() {
                return null;
            }
        };

        task.fork();
        task.join();
    }

    @Transactional
    private void processRecord(RawDataEntity record) {
        try {
            // Fetch analysis data in bulk before processing
            AnalysisEntity analysis = fetchAnalysisForRecord(record);

            if (analysis != null && analysis.getTradePrice().equals(record.getTradePrice())) {
                record.setRecordStatus("DUPLICATE");
            } else {
                record.setRecordStatus("PROCESSED");
                saveNewAnalysisEntity(record);
            }

            rawDataRepository.saveAndFlush(record);

            int count = processedCount.incrementAndGet();
            System.out.println("Processed record number: " + count);
        } catch (Exception e) {
            record.setRecordStatus("PROCESS_ERROR");
            rawDataRepository.saveAndFlush(record);
        }
    }

    private AnalysisEntity fetchAnalysisForRecord(RawDataEntity record) {
        AnalysisEntityId analysisId = new AnalysisEntityId(record.getTradeDate(), record.getSymbol());
        return analysisRepository.findById(analysisId).orElse(null);
    }

    private void saveNewAnalysisEntity(RawDataEntity record) {
        AnalysisEntityId analysisId = new AnalysisEntityId(record.getTradeDate(), record.getSymbol());
        AnalysisEntity analysis = new AnalysisEntity();
        analysis.setId(analysisId);
        analysis.setTradePrice(record.getTradePrice());
        analysisRepository.save(analysis);
    }

    public int getProcessedCount() {
        return processedCount.get();
    }
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
import java.util.concurrent.*;
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

    private static final int PAGE_SIZE = 1000;

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
                Pageable pageable = PageRequest.of(pageNumber, PAGE_SIZE);
                page = rawDataRepository.findByBatchNameAndRecordStatus(batch.getBatchName(), "UNPROCESSED", pageable);

                // Process each page in parallel using ForkJoinPool
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

    // Define RecursiveAction for processing a batch of records in parallel
    private class RecordProcessingTask extends RecursiveAction {
        private final List<RawDataEntity> records;

        public RecordProcessingTask(List<RawDataEntity> records) {
            this.records = records;
        }

        @Override
        @Transactional
        protected void compute() {
            // Process records in parallel
            records.parallelStream().forEach(this::processRecord);
        }

        private void processRecord(RawDataEntity record) {
            try {
                AnalysisEntity analysis = fetchAnalysisForRecord(record);

                if (analysis != null && analysis.getTradePrice().equals(record.getTradePrice())) {
                    record.setRecordStatus("DUPLICATE");
                } else {
                    record.setRecordStatus("PROCESSED");
                    saveNewAnalysisEntity(record);
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

    private AnalysisEntity fetchAnalysisForRecord(RawDataEntity record) {
        AnalysisEntityId analysisId = new AnalysisEntityId(record.getTradeDate(), record.getSymbol());
        return analysisRepository.findById(analysisId).orElse(null);
    }

    private void saveNewAnalysisEntity(RawDataEntity record) {
        AnalysisEntityId analysisId = new AnalysisEntityId(record.getTradeDate(), record.getSymbol());
        AnalysisEntity analysis = new AnalysisEntity();
        analysis.setId(analysisId);
        analysis.setTradePrice(record.getTradePrice());
        analysisRepository.save(analysis);
    }

    public int getProcessedCount() {
        return processedCount.get();
    }
}
```
