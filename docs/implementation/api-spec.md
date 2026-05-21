# API Specification

> **OUTDATED — needs a full rebuild before implementation.** This
> spec was written before the
> [Graph Model](../primitive/graph-model.md) was designed; nothing
> below should be treated as authoritative. A full rebuild needs
> to cover at minimum:
>
> - The current node catalog (User, Collective, Post, Comment,
>   Chat, ChatMessage, Item, Hashtag, Proposal, Network) plus
>   junction nodes (ChatMember, CollectiveMember, ItemOwnership) —
>   see [nodes.md](../primitive/nodes.md).
> - The uniform `(dim1, dim2)` tensor edge model — no per-action
>   relationship types like "follow" or "like" — and the
>   append-only layer semantics (no "unlike" or "delete"
>   operations); see
>   [graph-model.md](../primitive/graph-model.md) and
>   [layers.md](../primitive/layers.md).
> - Governance flows (Proposals, Shape A / Shape B votes,
>   tally-time eligibility) — see
>   [governance.md](../primitive/governance.md).
> - Per-field moderation targets and the redaction cascade — see
>   [moderation.md](../instances/moderation.md).
> - Auth flows (invitation acceptance, sessions, token rotation) —
>   see [auth.md](auth.md).
>
> Until that rebuild lands, treat any GraphQL fragments below as
> historical reference, not as a contract.

The API is a GraphQL endpoint served by Axum + async-graphql.

- **Endpoint**: `POST /graphql`
- **Playground**: `GET /playground` (dev mode only)
- **Health check**: `GET /health`

---

## GraphQL Schema

```graphql
scalar DateTime
scalar UUID

# ─── Types ────────────────────────────────────────────────────────────────────

type User {
  id:             UUID!
  username:       String!
  displayName:    String!
  bio:            String
  avatarUrl:      String
  websiteUrl:     String
  createdAt:      DateTime!

  # Resolved from graph
  followerCount:  Int!
  followingCount: Int!
  isFollowedByMe: Boolean!  # requires auth context
}

type Post {
  id:               UUID!
  author:           User!
  content:          String!
  mediaAttachments: [MediaAttachment!]!
  hashtags:         [String!]!
  createdAt:        DateTime!

  # Resolved from graph
  likeCount:        Int!
  commentCount:     Int!
  isLikedByMe:      Boolean!  # requires auth context
  comments:         [Comment!]!
}

type MediaAttachment {
  id:        UUID!
  url:       String!
  mimeType:  String!
  sizeBytes: Int
  altText:   String
}

type Comment {
  id:        UUID!
  author:    User!
  content:   String!
  createdAt: DateTime!
  replies:   [Comment!]!
}

# Feed item enriched with graph context
type FeedPost {
  post:               Post!
  degreeOfSeparation: Int!     # hops from viewing user to post author
  mutualFollowers:    [User!]! # viewing user's followers who also follow the author
}

# Graph path between users
type UserPath {
  users:   [User!]!
  degrees: Int!
}

# ─── Queries ──────────────────────────────────────────────────────────────────

type Query {
  # User lookups
  user(id: UUID, username: String): User
  me: User!  # requires auth

  # Personalized feed — ordered by graph topology + recency
  feed(limit: Int = 20, offset: Int = 0): [FeedPost!]!

  # Social graph traversal
  following(userId: UUID!, limit: Int = 50, offset: Int = 0): [User!]!
  followers(userId: UUID!, limit: Int = 50, offset: Int = 0): [User!]!
  mutualConnections(userId: UUID!): [User!]!
  suggestedUsers(limit: Int = 10): [User!]!

  # Content
  post(id: UUID!): Post
  userPosts(userId: UUID!, limit: Int = 20, offset: Int = 0): [Post!]!
  hashtagFeed(hashtag: String!, limit: Int = 20, offset: Int = 0): [Post!]!

  # Graph exploration
  shortestPath(fromUserId: UUID!, toUserId: UUID!): UserPath
  degreesOfSeparation(fromUserId: UUID!, toUserId: UUID!): Int
}

# ─── Mutations ────────────────────────────────────────────────────────────────

type Mutation {
  # Social graph
  followUser(userId: UUID!):   Boolean!
  unfollowUser(userId: UUID!): Boolean!
  blockUser(userId: UUID!):    Boolean!

  # Content
  createPost(input: CreatePostInput!):       Post!
  deletePost(postId: UUID!):                 Boolean!
  likePost(postId: UUID!):                   Boolean!
  unlikePost(postId: UUID!):                 Boolean!
  createComment(input: CreateCommentInput!): Comment!
  deleteComment(commentId: UUID!):           Boolean!

  # Profile
  updateProfile(input: UpdateProfileInput!): User!
}

# ─── Inputs ───────────────────────────────────────────────────────────────────

input CreatePostInput {
  content:   String!
  hashtags:  [String!]
  mediaUrls: [MediaInput!]
}

input MediaInput {
  url:      String!
  mimeType: String!
  altText:  String
}

input CreateCommentInput {
  postId:          UUID!
  content:         String!
  parentCommentId: UUID  # null = top-level comment
}

input UpdateProfileInput {
  displayName: String
  bio:         String
  avatarUrl:   String
  websiteUrl:  String
}
```

---

## Key Design Decisions

### Why GraphQL?

A social network's data is deeply relational and highly variable per view — a feed item needs user data, post data, like counts, and mutual follower context all at once. GraphQL lets clients request exactly what they need in a single round trip, avoiding N+1 fetch chains and over-fetching.

### Field-level resolver strategy

Graph-derived fields (`followerCount`, `likeCount`, `isLikedByMe`, etc.) are resolved lazily by async-graphql. If a client doesn't request them, the Cypher query never runs. This matters because not every query needs graph data.

### Pagination

`feed`, `userPosts`, `following`, `followers`, and `hashtagFeed` use offset pagination for the initial implementation. Cursor-based pagination (more efficient for real-time feeds) can be added later.

### Authentication

Auth is stubbed in this (outdated) schema. The `me` query and mutations requiring auth context will return an error until auth is implemented. The auth model — registration via invitation, email verification, JWT access + Postgres-backed refresh tokens, session listing, rate limiting — is specified in [auth.md](auth.md).

---

## Example Queries

### Get personalized feed
```graphql
query Feed {
  feed(limit: 10) {
    degreeOfSeparation
    mutualFollowers {
      username
      avatarUrl
    }
    post {
      id
      content
      createdAt
      likeCount
      author {
        username
        displayName
        avatarUrl
      }
      hashtags
      mediaAttachments {
        url
        mimeType
        altText
      }
    }
  }
}
```

### Find path between users
```graphql
query Path {
  shortestPath(fromUserId: "...", toUserId: "...") {
    degrees
    users {
      username
      avatarUrl
    }
  }
}
```

### Follow a user
```graphql
mutation Follow {
  followUser(userId: "...")
}
```
