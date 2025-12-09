# Conventions - Backend

## File Structure
- `src/resolvers/query/` - Query resolvers
- `src/resolvers/mutation/` - Mutation resolvers
- `src/services/` - Business logic
- `src/jobs/` - Step Function handlers
- `src/utils/` - Utilities

## Resolver Pattern
```typescript
export async function handler(event: AppSyncResolverEvent<Args>) {
  const userId = requireAuth(event)
  const service = new Service()
  return await service.method(userId, event.arguments)
}
```

## Error Handling
- Return user-friendly messages
- Log errors to CloudWatch
- Use typed errors from utils
