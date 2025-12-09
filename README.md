# backend

> GraphQL API, Lambda resolvers, and business logic for Kaze no Manga

## Overview

This repository contains the complete backend implementation for Kaze no Manga, including AWS AppSync GraphQL API, Lambda resolvers, Step Functions for background jobs, and notification services.

## Features

- 🔷 **GraphQL API**: AWS AppSync with type-safe schema
- ⚡ **Lambda Resolvers**: Node.js Lambda functions per resolver
- 📝 **VTL Templates**: Velocity templates for simple queries
- 🔄 **Direct Resolvers**: AppSync → DynamoDB for cache
- ⏰ **Background Jobs**: Step Functions for daily chapter checks
- 📬 **Notifications**: SES (email) + SNS (push)
- 🔐 **Authentication**: AWS Cognito integration
- 📊 **Monitoring**: CloudWatch logs and metrics

## Tech Stack

- **API**: AWS AppSync (GraphQL)
- **Compute**: AWS Lambda (Node.js 20.x)
- **Jobs**: AWS Step Functions + EventBridge
- **Queue**: AWS SQS
- **Notifications**: AWS SES + SNS
- **Auth**: AWS Cognito
- **Database**: RDS Postgres + DynamoDB (via `database` repo)
- **Scraper**: `@kaze/scraper` package
- **Models**: `@kaze/models` package

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    AppSync GraphQL                        │
│              (schema.graphql + resolvers)                 │
└──────────────────────────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌─────────┐     ┌─────────┐     ┌─────────┐
    │  VTL    │     │ Lambda  │     │ Direct  │
    │Template │     │Resolver │     │Resolver │
    └─────────┘     └─────────┘     └─────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌─────────┐     ┌─────────┐     ┌─────────┐
    │   RDS   │     │DynamoDB │     │   SQS   │
    │Postgres │     │ (Cache) │     │ (Jobs)  │
    └─────────┘     └─────────┘     └─────────┘
                                          │
                                          ▼
                                    ┌─────────┐
                                    │  Step   │
                                    │Function │
                                    └─────────┘
```

## GraphQL Schema

### Queries

```graphql
type Query {
  # Manga
  searchManga(query: String!): [Manga!]!
  getManga(id: ID!): Manga
  getChapters(mangaId: ID!): [Chapter!]!
  getChapter(id: ID!): Chapter
  
  # Library
  getLibrary: [LibraryEntry!]!
  getLibraryEntry(mangaId: ID!): LibraryEntry
  getReadingHistory(limit: Int = 50): [ReadingHistory!]!
  
  # User
  getProfile: User!
  getNotifications(unreadOnly: Boolean = false): [Notification!]!
  
  # Discovery
  getPopularManga(limit: Int = 20): [Manga!]!
  getRecentlyAddedManga(limit: Int = 20): [Manga!]!
  getRecommendations(limit: Int = 10): [Manga!]!
}
```

### Mutations

```graphql
type Mutation {
  # Library
  addToLibrary(mangaId: ID!): LibraryEntry!
  updateLibraryEntry(mangaId: ID!, input: LibraryEntryInput!): LibraryEntry!
  removeFromLibrary(mangaId: ID!): Boolean!
  
  # Progress
  updateProgress(mangaId: ID!, chapterId: ID!): LibraryEntry!
  markChapterAsRead(chapterId: ID!, completed: Boolean = true): ReadingHistory!
  markChaptersAsRead(chapterIds: [ID!]!): [ReadingHistory!]!
  
  # User
  updateProfile(input: UserProfileInput!): User!
  updatePreferences(input: UserPreferencesInput!): User!
  
  # Notifications
  markNotificationAsRead(id: ID!): Notification!
  markAllNotificationsAsRead: Int!
  
  # External Sync
  syncWithAniList(token: String!): SyncResult!
  syncWithMyAnimeList(token: String!): SyncResult!
}
```

### Subscriptions (Future)

```graphql
type Subscription {
  onNewChapter(mangaId: ID!): Chapter!
  onNotification: Notification!
}
```

## Lambda Resolvers

### Structure

```
src/
├── resolvers/
│   ├── query/
│   │   ├── searchManga.ts
│   │   ├── getManga.ts
│   │   ├── getLibrary.ts
│   │   └── ...
│   ├── mutation/
│   │   ├── addToLibrary.ts
│   │   ├── updateProgress.ts
│   │   ├── markChapterAsRead.ts
│   │   └── ...
│   └── field/
│       ├── Manga.chapters.ts
│       └── User.library.ts
├── services/
│   ├── manga.service.ts
│   ├── library.service.ts
│   ├── notification.service.ts
│   └── sync.service.ts
├── jobs/
│   ├── checkNewChapters.ts
│   ├── sendNotifications.ts
│   └── cleanupOldData.ts
└── utils/
    ├── auth.ts
    ├── errors.ts
    └── logger.ts
```

### Example Resolver

```typescript
// src/resolvers/mutation/addToLibrary.ts
import { AppSyncResolverEvent } from 'aws-lambda'
import { LibraryService } from '../../services/library.service'
import { requireAuth } from '../../utils/auth'

export async function handler(event: AppSyncResolverEvent<{ mangaId: string }>) {
  const userId = requireAuth(event)
  const { mangaId } = event.arguments
  
  const libraryService = new LibraryService()
  const entry = await libraryService.addToLibrary(userId, mangaId)
  
  return entry
}
```

### Example Service

```typescript
// src/services/library.service.ts
import { db } from 'database'
import { userLibrary, manga } from 'database/schema'
import { eq, and } from 'drizzle-orm'

export class LibraryService {
  async addToLibrary(userId: string, mangaId: string) {
    // Check if manga exists
    const mangaExists = await db.select().from(manga).where(eq(manga.id, mangaId))
    if (!mangaExists.length) {
      throw new Error('Manga not found')
    }
    
    // Check if already in library
    const existing = await db.select()
      .from(userLibrary)
      .where(and(
        eq(userLibrary.userId, userId),
        eq(userLibrary.mangaId, mangaId)
      ))
    
    if (existing.length) {
      return existing[0]
    }
    
    // Add to library
    const [entry] = await db.insert(userLibrary).values({
      userId,
      mangaId,
      status: 'plan_to_read',
      addedAt: new Date(),
      updatedAt: new Date(),
    }).returning()
    
    return entry
  }
  
  async updateProgress(userId: string, mangaId: string, chapterId: string) {
    // Implementation
  }
  
  async removeFromLibrary(userId: string, mangaId: string) {
    // Implementation
  }
}
```

## VTL Templates

For simple queries that don't need Lambda:

```velocity
## src/vtl/getLibrary.vtl
{
  "version": "2018-05-29",
  "operation": "Query",
  "query": {
    "expression": "user_id = :userId",
    "expressionValues": {
      ":userId": $util.dynamodb.toDynamoDBJson($ctx.identity.sub)
    }
  }
}
```

## Background Jobs

### Step Functions

```typescript
// src/jobs/checkNewChapters.ts
import { StepFunctionEvent } from 'aws-lambda'
import { db } from 'database'
import { manga, chapters, userLibrary } from 'database/schema'
import { createScraper } from '@kaze/scraper'
import { NotificationService } from '../services/notification.service'

export async function handler(event: StepFunctionEvent) {
  // Get all manga that need checking
  const mangaToCheck = await db.select()
    .from(manga)
    .where(/* priority logic */)
  
  for (const m of mangaToCheck) {
    // Get latest chapter from DB
    const latestChapter = await db.select()
      .from(chapters)
      .where(eq(chapters.mangaId, m.id))
      .orderBy(desc(chapters.number))
      .limit(1)
    
    // Check source for new chapters
    const scraper = createScraper(m.source)
    const sourceChapters = await scraper.getChapters(m.sourceId)
    
    // Find new chapters
    const newChapters = sourceChapters.filter(
      ch => ch.number > (latestChapter[0]?.number || 0)
    )
    
    if (newChapters.length > 0) {
      // Save new chapters
      await db.insert(chapters).values(newChapters)
      
      // Notify users
      const notificationService = new NotificationService()
      await notificationService.notifyNewChapters(m.id, newChapters)
    }
  }
}
```

### EventBridge Schedule

```typescript
// Triggered daily at 00:00 UTC
{
  "schedule": "cron(0 0 * * ? *)",
  "target": "CheckNewChaptersStepFunction"
}
```

## Notifications

### Email (SES)

```typescript
// src/services/notification.service.ts
import { SESClient, SendEmailCommand } from '@aws-sdk/client-ses'

export class NotificationService {
  private ses = new SESClient({})
  
  async sendNewChapterEmail(userId: string, manga: Manga, chapters: Chapter[]) {
    const user = await this.getUser(userId)
    
    if (!user.preferences.notifications.email) {
      return
    }
    
    const html = this.renderEmailTemplate('new-chapter', {
      manga,
      chapters,
      user,
    })
    
    await this.ses.send(new SendEmailCommand({
      Source: 'notifications@kazenomanga.com',
      Destination: { ToAddresses: [user.email] },
      Message: {
        Subject: { Data: `New chapters for ${manga.title}` },
        Body: { Html: { Data: html } },
      },
    }))
  }
}
```

### Push (SNS)

```typescript
// src/services/notification.service.ts
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns'

export class NotificationService {
  private sns = new SNSClient({})
  
  async sendPushNotification(userId: string, manga: Manga, chapters: Chapter[]) {
    const user = await this.getUser(userId)
    
    if (!user.preferences.notifications.push) {
      return
    }
    
    // Get user's device tokens from DynamoDB
    const tokens = await this.getUserDeviceTokens(userId)
    
    for (const token of tokens) {
      await this.sns.send(new PublishCommand({
        TargetArn: token.arn,
        Message: JSON.stringify({
          notification: {
            title: `New chapters for ${manga.title}`,
            body: `${chapters.length} new chapter(s) available`,
          },
          data: {
            mangaId: manga.id,
            chapterIds: chapters.map(ch => ch.id),
          },
        }),
      }))
    }
  }
}
```

## Authentication

```typescript
// src/utils/auth.ts
import { AppSyncResolverEvent } from 'aws-lambda'

export function requireAuth(event: AppSyncResolverEvent<any>): string {
  const userId = event.identity?.sub
  
  if (!userId) {
    throw new Error('Unauthorized')
  }
  
  return userId
}

export function optionalAuth(event: AppSyncResolverEvent<any>): string | null {
  return event.identity?.sub || null
}
```

## Error Handling

```typescript
// src/utils/errors.ts
export class AppError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number = 400
  ) {
    super(message)
    this.name = 'AppError'
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 'NOT_FOUND', 404)
  }
}

export class UnauthorizedError extends AppError {
  constructor() {
    super('Unauthorized', 'UNAUTHORIZED', 401)
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 'VALIDATION_ERROR', 400)
  }
}
```

## Development

```bash
# Install dependencies
npm install

# Build
npm run build

# Test
npm test

# Test specific resolver
npm test -- addToLibrary

# Deploy (via CDK in infrastructure repo)
cd ../infrastructure
npm run deploy
```

## Environment Variables

```bash
# .env
DATABASE_URL="postgresql://..."
DYNAMODB_TABLE_NAME="kaze-cache"
SQS_QUEUE_URL="https://sqs.us-east-1.amazonaws.com/..."
SES_FROM_EMAIL="notifications@kazenomanga.com"
SNS_PLATFORM_APPLICATION_ARN="arn:aws:sns:..."
```

## Testing

```typescript
// tests/resolvers/addToLibrary.test.ts
import { handler } from '../../src/resolvers/mutation/addToLibrary'

describe('addToLibrary', () => {
  it('should add manga to library', async () => {
    const event = {
      identity: { sub: 'user-123' },
      arguments: { mangaId: 'manga-456' },
    }
    
    const result = await handler(event as any)
    
    expect(result).toMatchObject({
      userId: 'user-123',
      mangaId: 'manga-456',
      status: 'plan_to_read',
    })
  })
  
  it('should throw if manga not found', async () => {
    const event = {
      identity: { sub: 'user-123' },
      arguments: { mangaId: 'invalid' },
    }
    
    await expect(handler(event as any)).rejects.toThrow('Manga not found')
  })
})
```

## Deployment

Deployed via AWS CDK (see `infrastructure` repo):

```typescript
// In infrastructure repo
const api = new appsync.GraphqlApi(this, 'KazeApi', {
  name: 'kaze-no-manga-api',
  schema: appsync.SchemaFile.fromAsset('path/to/schema.graphql'),
  authorizationConfig: {
    defaultAuthorization: {
      authorizationType: appsync.AuthorizationType.USER_POOL,
      userPoolConfig: { userPool },
    },
  },
})

// Add Lambda resolver
const addToLibraryResolver = new appsync.Resolver(this, 'AddToLibraryResolver', {
  api,
  typeName: 'Mutation',
  fieldName: 'addToLibrary',
  dataSource: api.addLambdaDataSource('AddToLibraryDS', addToLibraryLambda),
})
```

## Best Practices

1. **Keep resolvers thin** - Move logic to services
2. **Use VTL for simple queries** - Avoid Lambda overhead
3. **Cache aggressively** - Use DynamoDB for hot data
4. **Handle errors gracefully** - Return user-friendly messages
5. **Log everything** - Use structured logging
6. **Monitor performance** - CloudWatch metrics and alarms
7. **Test thoroughly** - Unit + integration tests

## License

MIT License - see [LICENSE](LICENSE) for details.

---

**Part of the [Kaze no Manga](https://github.com/kaze-no-manga) project**
