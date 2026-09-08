# InkWell — Social Blogging Database in MongoDB

A four-collection MongoDB data model for a social blogging platform, covering
schema design, CRUD operations, queries, sorting, and engagement analytics.

## Overview

InkWell is a small blogging platform where writers publish posts under
categories, and readers leave comments. This project designs the collections,
loads the platform's existing content, and builds the everyday operations
InkWell needs — publishing posts, updating profiles, moderating content, and
answering questions like "which author is getting the most engagement?"

**Environment:** `mongosh` connected to a local or Atlas MongoDB instance.

## Data Model

Four collections:

| Collection | Purpose |
|---|---|
| `categories` | The topics InkWell publishes under |
| `users` | Registered writers, with a running list of posts they've authored |
| `posts` | Every published article — author, category, like count |
| `comments` | Reader comments, linked to the post and user who wrote them |

**Design note:** `comments` is its own collection rather than an array
embedded in `posts` — a popular post could accumulate hundreds of comments,
and keeping comments separate keeps each post document small and quick to
load.

### Schema reference

**categories**
```js
{ _id: "CAT01", name: "Tech", description: "Software, gadgets and how-to guides" }
```

**users**
```js
{
  _id: "U01", name: "Ananya Rao", email: "ananya@example.com",
  bio: "Backend engineer writing about databases and cloud.",
  posts: ["P01", "P06"]
}
```

**posts**
```js
{
  _id: "P01", title: "Getting Started with MongoDB", authorId: "U01",
  category: "Tech", likes: 120, publishedDate: ISODate("2025-01-05")
}
```

**comments**
```js
{
  _id: "C01", postId: "P01", userId: "U02",
  commentText: "Great intro, very helpful!", commentDate: ISODate("2025-01-06")
}
```

## Build Phases

1. **Environment & Schema Setup** — create `inkwell_db` and load seed data
   (5 categories, 5 users, 10 posts, 12 comments).
2. **Maintaining Relationships** — sync each user's `posts` array with the
   posts that actually reference them as author.
3. **CRUD Operations** — add a user, publish a post, bump like counts,
   update a bio, and remove a post for a policy violation.
4. **Querying & Filtering** — category/likes filters, per-author posts,
   date-range posts, field projection, comments on a specific post.
5. **Sorting** — most liked posts, most recently published posts.
6. **Aggregation & Analytics** — total likes, average likes, top author,
   total comments, monthly publishing trend.

## Full Script

Copy this whole block into `mongosh`, or paste it phase by phase. It's
written plainly — no loops or advanced JS — so every line shows exactly
what it does.

```javascript
/* ===================================================================
   InkWell — Social Blogging Database in MongoDB
   Run these in mongosh, in order, phase by phase.
=================================================================== */

/* ===================================================================
   PHASE 1 — Environment & Schema Setup
=================================================================== */

use inkwell_db

db.categories.insertMany([
  { _id: "CAT01", name: "Tech", description: "Software, gadgets and how-to guides" },
  { _id: "CAT02", name: "Travel", description: "Destinations, guides and travel stories" },
  { _id: "CAT03", name: "Food", description: "Recipes and cooking tips" },
  { _id: "CAT04", name: "Lifestyle", description: "Everyday living and wellness" },
  { _id: "CAT05", name: "Finance", description: "Money, budgeting and investing" }
])

db.users.insertMany([
  { _id: "U01", name: "Ananya Rao", email: "ananya@example.com", bio: "Backend engineer writing about databases and cloud.", posts: [] },
  { _id: "U02", name: "Rohan Mehta", email: "rohan@example.com", bio: "Travel enthusiast documenting trips around the world.", posts: [] },
  { _id: "U03", name: "Kavya Singh", email: "kavya@example.com", bio: "Home cook sharing quick and easy recipes.", posts: [] },
  { _id: "U04", name: "Ishaan Verma", email: "ishaan@example.com", bio: "Minimalist writing about intentional living.", posts: [] },
  { _id: "U05", name: "Meera Nair", email: "meera@example.com", bio: "Personal finance blogger and budgeting coach.", posts: [] }
])

db.posts.insertMany([
  { _id: "P01", title: "Getting Started with MongoDB", authorId: "U01", category: "Tech", likes: 120, publishedDate: ISODate("2025-01-05") },
  { _id: "P02", title: "Top 5 Travel Destinations in 2025", authorId: "U02", category: "Travel", likes: 85, publishedDate: ISODate("2025-01-10") },
  { _id: "P03", title: "Easy Weeknight Pasta Recipes", authorId: "U03", category: "Food", likes: 60, publishedDate: ISODate("2025-01-15") },
  { _id: "P04", title: "Minimalist Living Tips", authorId: "U04", category: "Lifestyle", likes: 45, publishedDate: ISODate("2025-01-20") },
  { _id: "P05", title: "Understanding Mutual Funds", authorId: "U05", category: "Finance", likes: 95, publishedDate: ISODate("2025-02-01") },
  { _id: "P06", title: "Advanced JavaScript Patterns", authorId: "U01", category: "Tech", likes: 150, publishedDate: ISODate("2025-02-10") },
  { _id: "P07", title: "Backpacking Through Europe", authorId: "U02", category: "Travel", likes: 110, publishedDate: ISODate("2025-02-18") },
  { _id: "P08", title: "Healthy Breakfast Ideas", authorId: "U03", category: "Food", likes: 70, publishedDate: ISODate("2025-03-02") },
  { _id: "P09", title: "Budgeting for Beginners", authorId: "U05", category: "Finance", likes: 130, publishedDate: ISODate("2025-03-10") },
  { _id: "P10", title: "Digital Detox: A Weekend Guide", authorId: "U04", category: "Lifestyle", likes: 55, publishedDate: ISODate("2025-03-20") }
])

db.comments.insertMany([
  { _id: "C01", postId: "P01", userId: "U02", commentText: "Great intro, very helpful!", commentDate: ISODate("2025-01-06") },
  { _id: "C02", postId: "P01", userId: "U03", commentText: "Thanks for this guide.", commentDate: ISODate("2025-01-07") },
  { _id: "C03", postId: "P02", userId: "U01", commentText: "Adding these to my bucket list!", commentDate: ISODate("2025-01-11") },
  { _id: "C04", postId: "P03", userId: "U04", commentText: "Tried the pasta recipe, delicious!", commentDate: ISODate("2025-01-16") },
  { _id: "C05", postId: "P05", userId: "U02", commentText: "Very informative, thanks!", commentDate: ISODate("2025-02-02") },
  { _id: "C06", postId: "P06", userId: "U03", commentText: "These patterns changed how I code.", commentDate: ISODate("2025-02-11") },
  { _id: "C07", postId: "P06", userId: "U05", commentText: "Bookmarking this for later.", commentDate: ISODate("2025-02-12") },
  { _id: "C08", postId: "P07", userId: "U04", commentText: "Europe is on my list now!", commentDate: ISODate("2025-02-19") },
  { _id: "C09", postId: "P09", userId: "U01", commentText: "Great budgeting tips!", commentDate: ISODate("2025-03-11") },
  { _id: "C10", postId: "P09", userId: "U03", commentText: "Very practical advice.", commentDate: ISODate("2025-03-12") },
  { _id: "C11", postId: "P09", userId: "U04", commentText: "Needed this, thank you!", commentDate: ISODate("2025-03-13") },
  { _id: "C12", postId: "P10", userId: "U02", commentText: "Doing a digital detox this weekend!", commentDate: ISODate("2025-03-21") }
])

// Verify counts: 5 categories, 5 users, 10 posts, 12 comments
db.categories.countDocuments()
db.users.countDocuments()
db.posts.countDocuments()
db.comments.countDocuments()


/* ===================================================================
   PHASE 2 — Maintaining Relationships
   We already know from the seed data which posts belong to which
   author, so we just set each user's "posts" array directly.
   (No loop needed — there are only 5 users.)

   U01 -> P01, P06
   U02 -> P02, P07
   U03 -> P03, P08
   U04 -> P04, P10
   U05 -> P05, P09
=================================================================== */

db.users.updateOne({ _id: "U01" }, { $set: { posts: ["P01", "P06"] } })
db.users.updateOne({ _id: "U02" }, { $set: { posts: ["P02", "P07"] } })
db.users.updateOne({ _id: "U03" }, { $set: { posts: ["P03", "P08"] } })
db.users.updateOne({ _id: "U04" }, { $set: { posts: ["P04", "P10"] } })
db.users.updateOne({ _id: "U05" }, { $set: { posts: ["P05", "P09"] } })

// Verify: U01's posts array should contain exactly ["P01", "P06"]
db.users.findOne({ _id: "U01" }).posts


/* ===================================================================
   PHASE 3 — CRUD Operations
=================================================================== */

// 1. Add a new user
db.users.insertOne({
  _id: "U06",
  name: "Diya Patel",
  email: "diya@example.com",
  bio: "Food blogger and recipe developer.",
  posts: []
})

// 2. Diya publishes a new post
db.posts.insertOne({
  _id: "P11",
  title: "5-Minute Breakfast Smoothies",
  authorId: "U06",
  category: "Food",
  likes: 0,
  publishedDate: ISODate("2025-03-28")
})

// Update Diya's posts array to include it
db.users.updateOne(
  { _id: "U06" },
  { $set: { posts: ["P11"] } }
)

// 3. Two posts picked up new likes
// P01: 120 + 15 = 135
db.posts.updateOne({ _id: "P01" }, { $set: { likes: 135 } })
// P05: 95 + 20 = 115
db.posts.updateOne({ _id: "P05" }, { $set: { likes: 115 } })

// 4. U02 updated their bio
db.users.updateOne(
  { _id: "U02" },
  { $set: { bio: "Full-time travel blogger and photographer." } }
)

// 5. P04 removed for a policy violation — delete the post and remove
// the reference from U04's posts array
db.posts.deleteOne({ _id: "P04" })

// U04's posts array becomes just ["P10"] after removing "P04"
db.users.updateOne(
  { _id: "U04" },
  { $set: { posts: ["P10"] } }
)

// Spot-check
db.posts.findOne({ _id: "P01" }).likes    // 135
db.posts.findOne({ _id: "P05" }).likes    // 115
db.users.findOne({ _id: "U06" }).posts    // ["P11"]
db.users.findOne({ _id: "U02" }).bio
db.posts.findOne({ _id: "P04" })          // null (deleted)
db.users.findOne({ _id: "U04" }).posts    // ["P10"]


/* ===================================================================
   PHASE 4 — Querying & Filtering
=================================================================== */

// 1. Tech posts with more than 100 likes
db.posts.find({ category: "Tech", likes: { $gt: 100 } })

// 2. All posts written by U01
db.posts.find({ authorId: "U01" })

// 3. Posts published in February 2025 (inclusive of the whole month)
db.posts.find({
  publishedDate: {
    $gte: ISODate("2025-02-01"),
    $lte: ISODate("2025-02-28")
  }
})

// 4. title and likes only — no _id
db.posts.find({}, { _id: 0, title: 1, likes: 1 })

// 5. All comments left on post P09
db.comments.find({ postId: "P09" })


/* ===================================================================
   PHASE 5 — Sorting
=================================================================== */

// 1. Top 5 most liked posts (1 = ascending, -1 = descending)
db.posts.find().sort({ likes: -1 }).limit(5)

// 2. 3 most recently published posts
db.posts.find().sort({ publishedDate: -1 }).limit(3)


/* ===================================================================
   PHASE 6 — Aggregation & Analytics
   Every pipeline below follows the same pattern:
     $group  -> add up or average the numbers you care about
   (No $unwind needed here — unlike BuildKart's orderItems, likes and
   comments already live as single fields on their own documents.)
=================================================================== */

// 1. Total likes across every post
db.posts.aggregate([
  { $group: {
      _id: null,
      totalLikes: { $sum: "$likes" }
  } }
])

// 2. Average number of likes per post
db.posts.aggregate([
  { $group: {
      _id: null,
      avgLikes: { $avg: "$likes" }
  } }
])

// 3. Author with the most total likes across their posts
db.posts.aggregate([
  { $group: {
      _id: "$authorId",
      totalLikes: { $sum: "$likes" }
  } },
  { $sort: { totalLikes: -1 } },
  { $limit: 1 }
])

// 4. Total number of comments left across all posts
db.comments.aggregate([
  { $group: {
      _id: null,
      totalComments: { $sum: 1 }
  } }
])

// 5. Posts published per month (publishing trend)
db.posts.aggregate([
  { $group: {
      _id: { year: { $year: "$publishedDate" }, month: { $month: "$publishedDate" } },
      postCount: { $sum: 1 }
  } },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])
```

## Phase 6 — Engagement Insights

Verified results from the aggregation pipelines:

- **Total likes:** 910 across all 10 posts (after Phase 3's updates and P04's removal)
- **Average likes per post:** 91
- **Top author:** U01 (Ananya Rao) — 285 total likes across "Getting Started
  with MongoDB" and "Advanced JavaScript Patterns"
- **Total comments:** 12 across the platform
- **Most-commented post:** P09 ("Budgeting for Beginners") with 3 comments
- **Publishing trend:** Jan 3 posts → Feb 3 posts → Mar 4 posts — steady
  and slightly increasing output

## How to Run

1. Start `mongosh` connected to your MongoDB instance.
2. Copy the **Full Script** section above and run each phase's commands in
   order, top to bottom.
3. Check the verification lines after each phase (document counts, spot
   checks on updated fields) before moving on — Phase 6's numbers depend on
   Phases 1–3 being correct.

## Notes

- Phase 4's February date filter uses `$gte`/`$lte` so the whole month is
  included, not just February 1st.
- Phase 6 doesn't need `$unwind` the way BuildKart's order analytics did —
  `likes` and comment documents are already flat fields, not nested arrays,
  so `$group` alone is enough.
- P04 had no comments referencing it in the seed data, so deleting it in
  Phase 3 doesn't require any comment cleanup.
