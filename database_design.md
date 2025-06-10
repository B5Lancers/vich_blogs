# Database Schema Fields and Relationships Guide

## Schema Overview & Relationships

```
auth.users (Supabase Auth)
    ↓ (1:1)
profiles
    ↓ (1:many)
posts ←→ categories (many:1)
    ↓ (many:many)
post_tags ←→ tags
    ↓ (1:many)
comments (self-referencing for replies)
post_views
post_likes
media
```

---

## 1. Profiles Table (User Management)

### Table: `profiles`

```sql
id UUID REFERENCES auth.users(id) ON DELETE CASCADE PRIMARY KEY,
username TEXT UNIQUE,
full_name TEXT,
avatar_url TEXT,
bio TEXT,
role user_role DEFAULT 'reader',
created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
```

### Field Breakdown

* **id**: References `auth.users(id)`, 1:1 relation
* **username**: Unique public handle (e.g., `/author/john-doe`)
* **full\_name**: Display name
* **avatar\_url**: Profile picture (Supabase Storage)
* **bio**: Short author biography
* **role**: Enum (`reader`, `author`, `editor`, `admin`)
* **created\_at / updated\_at**: Timestamps

### Relationships

* 1:1 with `auth.users`
* 1\:many with `posts`, `comments`, `post_likes`, `media`

---

## 2. Categories Table (Content Organization)

### Table: `categories`

```sql
id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
name TEXT NOT NULL UNIQUE,
slug TEXT NOT NULL UNIQUE,
description TEXT,
color TEXT DEFAULT '#6B7280',
created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
```

### Relationships

* 1\:many with `posts`

### Example Usage

```sql
SELECT p.*, c.name as category_name, c.color
FROM posts p
JOIN categories c ON p.category_id = c.id
WHERE c.slug = 'web-development' AND p.status = 'published';
```

---

## 3. Tags Table (Flexible Labeling)

### Table: `tags`

```sql
id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
name TEXT NOT NULL UNIQUE,
slug TEXT NOT NULL UNIQUE,
description TEXT,
created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
```

### Relationships

* many\:many with `posts` via `post_tags`

### Example Usage

```sql
SELECT p.*, array_agg(t.name) as tags
FROM posts p
JOIN post_tags pt ON p.id = pt.post_id
JOIN tags t ON pt.tag_id = t.id
WHERE t.slug = 'flutter'
GROUP BY p.id;
```

---

## 4. Posts Table (Core Content)

### Table: `posts`

```sql
id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
title TEXT NOT NULL,
slug TEXT NOT NULL UNIQUE,
excerpt TEXT,
content_md TEXT NOT NULL,
content_html TEXT,
author_id UUID REFERENCES profiles(id) ON DELETE CASCADE NOT NULL,
category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
status post_status DEFAULT 'draft',
featured BOOLEAN DEFAULT FALSE,
hero_image_url TEXT,
hero_image_alt TEXT,
meta_title TEXT,
meta_description TEXT,
published_at TIMESTAMP WITH TIME ZONE,
scheduled_at TIMESTAMP WITH TIME ZONE,
view_count INTEGER DEFAULT 0,
like_count INTEGER DEFAULT 0,
comment_count INTEGER DEFAULT 0,
reading_time_minutes INTEGER,
created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
search_vector tsvector GENERATED ALWAYS AS (...)
```

### Relationships

* author → `profiles`
* category → `categories`

### Example Usage

```sql
SELECT
  p.*,
  c.name as category_name,
  c.color as category_color,
  array_agg(t.name) as tags
FROM posts p
LEFT JOIN categories c ON p.category_id = c.id
LEFT JOIN post_tags pt ON p.id = pt.post_id
LEFT JOIN tags t ON pt.tag_id = t.id
WHERE p.slug = 'my-post-slug'
GROUP BY p.id, c.name, c.color;
```

---

## 5. Post\_Tags Table (Many-to-Many Junction)

### Table: `post_tags`

```sql
post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
tag_id UUID REFERENCES tags(id) ON DELETE CASCADE,
PRIMARY KEY (post_id, tag_id)
```

### Example Usage

```sql
-- Add tags
INSERT INTO post_tags (post_id, tag_id) VALUES (...);

-- Remove tag
DELETE FROM post_tags WHERE post_id = ... AND tag_id = ...;
```

---

## 6. Comments Table (User Engagement)

### Table: `comments`

```sql
id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
post_id UUID REFERENCES posts(id) ON DELETE CASCADE NOT NULL,
author_id UUID REFERENCES profiles(id) ON DELETE CASCADE,
parent_id UUID REFERENCES comments(id) ON DELETE CASCADE,
content TEXT NOT NULL,
author_name TEXT,
author_email TEXT,
is_approved BOOLEAN DEFAULT FALSE,
created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
```

### Relationships

* Self-referencing for replies
* Supports authenticated and anonymous comments

### Example Usage

```sql
SELECT
  c.*,
  pr.full_name,
  pr.avatar_url,
  CASE WHEN c.author_id IS NOT NULL THEN pr.full_name ELSE c.author_name END AS display_name
FROM comments c
LEFT JOIN profiles pr ON c.author_id = pr.id
WHERE c.post_id = 'post-uuid' AND c.is_approved = true
ORDER BY c.created_at;
```

---

## 7. Post\_Views Table (Analytics)

### Table: `post_views`

```sql
id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
post_id UUID REFERENCES posts(id) ON DELETE CASCADE NOT NULL,
user_id UUID REFERENCES profiles(id) ON DELETE SET NULL,
ip_address INET,
user_agent TEXT,
created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
```

### Example Usage

```sql
INSERT INTO post_views (post_id, user_id, ip_address, user_agent)
VALUES (...) ON CONFLICT DO NOTHING;
```

---

## 8. Post\_Likes Table (User Engagement)

### Table: `post_likes`

```sql
post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
user_id UUID REFERENCES profiles(id) ON DELETE CASCADE,
created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
PRIMARY KEY (post_id, user_id)
```

### Example Usage

```sql
-- Like
INSERT INTO post_likes (post_id, user_id) VALUES (...) ON CONFLICT DO NOTHING;
-- Unlike
DELETE FROM post_likes WHERE post_id = ... AND user_id = ...;
```

---

## 9. Media Table (Asset Management)

### Table: `media`

```sql
id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
filename TEXT NOT NULL,
original_filename TEXT NOT NULL,
file_path TEXT NOT NULL,
file_size INTEGER,
mime_type TEXT,
alt_text TEXT,
caption TEXT,
uploaded_by UUID REFERENCES profiles(id) ON DELETE SET NULL,
created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
```

---

## How Everything Works Together

### 1. Publishing Workflow

```sql
-- Create draft
INSERT INTO posts (...);
-- Add category/tags
UPDATE posts SET category_id = ...;
INSERT INTO post_tags (...);
-- Publish
UPDATE posts SET status = 'published', published_at = NOW() WHERE id = ...;
```

### 2. Reading Experience

```sql
SELECT
  p.*,
  pr.full_name as author_name,
  pr.avatar_url as author_avatar,
  pr.bio as author_bio,
  c.name as category_name,
  c.slug as category_slug,
  c.color as category_color,
  array_agg(DISTINCT t.name) as tags,
  COUNT(DISTINCT pl.user_id) as like_count,
  COUNT(DISTINCT co.id) as comment_count,
  EXISTS(
    SELECT 1 FROM post_likes WHERE post_id = p.id AND user_id = 'current-user-uuid'
  ) as user_liked
FROM posts p
JOIN profiles pr ON p.author_id = pr.id
LEFT JOIN categories c ON p.category_id = c.id
LEFT JOIN post_tags pt ON p.id = pt.post_id
LEFT JOIN tags t ON pt.tag_id = t.id
LEFT JOIN post_likes pl ON p.id = pl.post_id
LEFT JOIN comments co ON p.id = co.post_id AND co.is_approved = true
WHERE p.slug = 'post-slug' AND p.status = 'published'
GROUP BY p.id, pr.full_name, pr.avatar_url, pr.bio, c.name, c.slug, c.color;
```

### 3. Full-Text Search

```sql
SELECT p.*, pr.full_name as author_name,
       ts_rank(p.search_vector, plainto_tsquery('english', 'flutter tutorial')) as rank
FROM posts p
JOIN profiles pr ON p.author_id = pr.id
WHERE p.search_vector @@ plainto_tsquery('english', 'flutter tutorial')
  AND p.status = 'published'
ORDER BY rank DESC;
```

### 4. Analytics Dashboard

```sql
SELECT
  p.title,
  p.view_count,
  p.like_count,
  p.comment_count,
  COUNT(pv.id) as total_views_tracked,
  COUNT(DISTINCT pv.ip_address) as unique_visitors
FROM posts p
LEFT JOIN post_views pv ON p.id = pv.post_id
WHERE p.author_id = 'author-uuid'
GROUP BY p.id, p.title, p.view_count, p.like_count, p.comment_count
ORDER BY p.view_count DESC;
```
