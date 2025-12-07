# MongoDB Patterns

## Schema Design Philosophy

| SQL Approach | MongoDB Approach |
|--------------|------------------|
| Normalize data | Embed related data |
| Join at query time | Pre-join by embedding |
| Foreign keys | References (manual) |
| Schema enforced | Schema optional (use validation) |

## Embedding vs Referencing

### When to Embed

- Data accessed together (1:1, 1:few)
- Child doesn't exist without parent
- Updates are atomic
- Document size < 16MB

```javascript
// Embedded: User with addresses
{
  _id: ObjectId("..."),
  name: "John Doe",
  email: "john@example.com",
  addresses: [
    { type: "home", street: "123 Main St", city: "NYC" },
    { type: "work", street: "456 Office Blvd", city: "NYC" }
  ]
}
```

### When to Reference

- Data accessed independently
- Many-to-many relationships
- Large/unbounded arrays
- Shared across documents

```javascript
// Referenced: User and Orders
// users collection
{
  _id: ObjectId("user123"),
  name: "John Doe",
  email: "john@example.com"
}

// orders collection
{
  _id: ObjectId("order456"),
  userId: ObjectId("user123"),  // Reference
  items: [...],
  total: 99.99
}
```

## Common Patterns

### One-to-Many (Embedded)

```javascript
// Blog post with comments (few comments)
{
  _id: ObjectId("..."),
  title: "MongoDB Patterns",
  content: "...",
  comments: [
    { author: "Alice", text: "Great post!", createdAt: ISODate("...") },
    { author: "Bob", text: "Very helpful", createdAt: ISODate("...") }
  ]
}
```

### One-to-Many (Referenced)

```javascript
// Blog post with many comments
// posts collection
{
  _id: ObjectId("post123"),
  title: "MongoDB Patterns",
  content: "..."
}

// comments collection
{
  _id: ObjectId("..."),
  postId: ObjectId("post123"),
  author: "Alice",
  text: "Great post!",
  createdAt: ISODate("...")
}

// Query with $lookup
db.posts.aggregate([
  { $match: { _id: ObjectId("post123") } },
  {
    $lookup: {
      from: "comments",
      localField: "_id",
      foreignField: "postId",
      as: "comments"
    }
  }
]);
```

### Many-to-Many

```javascript
// Users and Roles
// users collection
{
  _id: ObjectId("user123"),
  name: "John",
  roleIds: [ObjectId("role1"), ObjectId("role2")]
}

// roles collection
{
  _id: ObjectId("role1"),
  name: "admin",
  permissions: ["read", "write", "delete"]
}

// Query user with roles
db.users.aggregate([
  { $match: { _id: ObjectId("user123") } },
  {
    $lookup: {
      from: "roles",
      localField: "roleIds",
      foreignField: "_id",
      as: "roles"
    }
  }
]);
```

### Polymorphic Pattern

```javascript
// Single collection for different content types
{
  _id: ObjectId("..."),
  type: "article",
  title: "...",
  content: "...",
  author: "..."
}

{
  _id: ObjectId("..."),
  type: "video",
  title: "...",
  url: "https://...",
  duration: 120
}

// Query by type
db.content.find({ type: "article" });
```

### Bucket Pattern (Time Series)

```javascript
// Instead of one document per measurement
// Group measurements into buckets
{
  _id: ObjectId("..."),
  sensorId: "sensor001",
  date: ISODate("2024-01-01"),
  measurements: [
    { time: ISODate("2024-01-01T00:00:00"), value: 23.5 },
    { time: ISODate("2024-01-01T00:01:00"), value: 23.6 },
    // ... up to ~200 per bucket
  ],
  count: 2,
  sum: 47.1,
  avg: 23.55
}
```

### Computed Pattern

```javascript
// Pre-compute expensive calculations
{
  _id: ObjectId("..."),
  title: "Product",
  reviews: [
    { rating: 5, text: "..." },
    { rating: 4, text: "..." }
  ],
  // Computed fields (update on review add/remove)
  reviewCount: 2,
  avgRating: 4.5,
  lastReviewAt: ISODate("...")
}
```

## Indexing

### Basic Indexes

```javascript
// Single field
db.users.createIndex({ email: 1 });  // Ascending
db.users.createIndex({ createdAt: -1 });  // Descending

// Compound
db.posts.createIndex({ userId: 1, createdAt: -1 });

// Unique
db.users.createIndex({ email: 1 }, { unique: true });

// Sparse (only index docs with field)
db.users.createIndex({ nickname: 1 }, { sparse: true });

// TTL (auto-delete after time)
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
```

### Text Index

```javascript
// Create text index
db.posts.createIndex({ title: "text", content: "text" });

// Search
db.posts.find({ $text: { $search: "mongodb patterns" } });

// With score
db.posts.find(
  { $text: { $search: "mongodb" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } });
```

### Partial Index

```javascript
// Only index active users
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { status: "active" } }
);
```

### Wildcard Index

```javascript
// Index all fields in subdocument
db.products.createIndex({ "attributes.$**": 1 });

// Query any attribute
db.products.find({ "attributes.color": "red" });
```

## Schema Validation

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "name", "createdAt"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        },
        name: {
          bsonType: "string",
          minLength: 2,
          maxLength: 100
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        },
        status: {
          enum: ["active", "inactive", "pending"]
        },
        createdAt: {
          bsonType: "date"
        }
      }
    }
  },
  validationLevel: "moderate",  // strict | moderate
  validationAction: "error"     // error | warn
});
```

## Aggregation Pipeline

### Basic Aggregation

```javascript
db.orders.aggregate([
  // Filter
  { $match: { status: "completed" } },

  // Group and calculate
  {
    $group: {
      _id: "$userId",
      totalOrders: { $sum: 1 },
      totalSpent: { $sum: "$total" },
      avgOrderValue: { $avg: "$total" }
    }
  },

  // Sort
  { $sort: { totalSpent: -1 } },

  // Limit
  { $limit: 10 },

  // Reshape output
  {
    $project: {
      userId: "$_id",
      totalOrders: 1,
      totalSpent: { $round: ["$totalSpent", 2] },
      avgOrderValue: { $round: ["$avgOrderValue", 2] }
    }
  }
]);
```

### Lookup (Join)

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
    }
  },
  { $unwind: "$user" },
  {
    $project: {
      orderId: "$_id",
      total: 1,
      userName: "$user.name",
      userEmail: "$user.email"
    }
  }
]);
```

### Faceted Search

```javascript
db.products.aggregate([
  { $match: { category: "electronics" } },
  {
    $facet: {
      // Results
      results: [
        { $sort: { price: 1 } },
        { $skip: 0 },
        { $limit: 20 }
      ],
      // Price ranges
      priceRanges: [
        {
          $bucket: {
            groupBy: "$price",
            boundaries: [0, 100, 500, 1000, Infinity],
            default: "Other",
            output: { count: { $sum: 1 } }
          }
        }
      ],
      // Brands
      brands: [
        { $group: { _id: "$brand", count: { $sum: 1 } } }
      ],
      // Total count
      totalCount: [{ $count: "count" }]
    }
  }
]);
```

## Transactions

```javascript
const session = client.startSession();

try {
  session.startTransaction();

  await db.accounts.updateOne(
    { _id: fromAccountId },
    { $inc: { balance: -amount } },
    { session }
  );

  await db.accounts.updateOne(
    { _id: toAccountId },
    { $inc: { balance: amount } },
    { session }
  );

  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
  throw error;
} finally {
  session.endSession();
}
```

## Best Practices

1. **Design for access patterns** - Know your queries before designing schema
2. **Embed when possible** - Reduces lookups, improves performance
3. **Avoid unbounded arrays** - Can hit 16MB limit, hard to query
4. **Index for your queries** - Check with `.explain("executionStats")`
5. **Use schema validation** - Enforce data integrity
6. **Use projections** - Only return needed fields
7. **Consider sharding early** - Choose good shard key
