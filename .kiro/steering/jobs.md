# Background Jobs - Backend

## Step Functions

### Daily Chapter Check
```typescript
// src/jobs/checkNewChapters.ts
export async function handler(event: StepFunctionEvent) {
  // 1. Get manga to check
  const mangaList = await getMangaToCheck()
  
  // 2. Check each manga
  for (const manga of mangaList) {
    const newChapters = await checkForNewChapters(manga)
    
    if (newChapters.length > 0) {
      await saveNewChapters(newChapters)
      await notifyUsers(manga, newChapters)
    }
  }
}
```

### EventBridge Schedule
```typescript
// Runs daily at 00:00 UTC
schedule: 'cron(0 0 * * ? *)'
```

## SQS Handlers
```typescript
// src/jobs/processScraperJob.ts
export async function handler(event: SQSEvent) {
  for (const record of event.Records) {
    const job = JSON.parse(record.body)
    await processJob(job)
  }
}
```
