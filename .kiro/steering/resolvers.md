# Resolvers - Backend

## Resolver Pattern

### Query Resolver
```typescript
// src/resolvers/query/getManga.ts
export async function handler(event: AppSyncResolverEvent<{ id: string }>) {
  const { id } = event.arguments
  
  const service = new MangaService()
  const manga = await service.getManga(id)
  
  if (!manga) {
    throw new NotFoundError('Manga')
  }
  
  return manga
}
```

### Mutation Resolver
```typescript
// src/resolvers/mutation/addToLibrary.ts
export async function handler(event: AppSyncResolverEvent<{ mangaId: string }>) {
  const userId = requireAuth(event)
  const { mangaId } = event.arguments
  
  const service = new LibraryService()
  return await service.addToLibrary(userId, mangaId)
}
```

## Service Pattern
```typescript
// src/services/manga.service.ts
export class MangaService {
  async getManga(id: string): Promise<Manga | null> {
    return db.query.manga.findFirst({
      where: eq(manga.id, id)
    })
  }
}
```

## Error Handling
```typescript
try {
  // Operation
} catch (error) {
  console.error('Error:', error)
  throw new AppError('User-friendly message', 'ERROR_CODE')
}
```
