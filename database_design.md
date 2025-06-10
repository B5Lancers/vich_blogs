
Database Schema Fields and Relationships Guide
Schema Overview & Relationships
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

1. Profiles Table (User Management)
Fields Explained
profiles (
  id UUID REFERENCES auth.users(id) ON DELETE CASCADE PRIMARY KEY,
  username TEXT UNIQUE,
  full_name TEXT,
  avatar_url TEXT,
  bio TEXT,
  role user_role DEFAULT 'reader',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
)

Field Breakdown:
id: Primary key that directly references Supabase's auth.users.id
Why: Creates a 1:1 relationship with authentication data
Usage: Every authenticated user automatically gets a profile
username: Unique identifier for public display
Why: URLs like /author/john-doe instead of UUID
Usage: SEO-friendly author pages, @mentions in comments
full_name: Display name for the user
Usage: Bylines on articles, comment author names
avatar_url: Profile picture URL (Supabase Storage)
Usage: Author avatars, comment author images
bio: Author biography
Usage: Author pages, article bylines
role: User permission level (reader, author, editor, admin)
reader: Can view published content, like, comment
author: Can create/edit own posts, upload media
editor: Can edit any post, moderate comments
admin: Full system access, user management
Relationships
1:1 with auth.users: Each auth user has exactly one profile
1:many with posts: Authors can write multiple posts
1:many with comments: Users can write multiple comments
1:many with post_likes: Users can like multiple posts
1:many with media: Users can upload multiple files

2. Categories Table (Content Organization)
Fields Explained
categories (
  id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  slug TEXT NOT NULL UNIQUE,
  description TEXT,
  color TEXT DEFAULT '#6B7280',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
)

Field Breakdown:
name: Human-readable category name
Example: "Web Development", "Machine Learning"
Usage: Category navigation, post organization
slug: URL-friendly version of name
Example: "web-development", "machine-learning"
Usage: SEO URLs like /category/web-development
description: Category explanation
Usage: Category pages, SEO meta descriptions
color: Hex color for UI theming
Usage: Category badges, visual distinction
Relationships
1:many with posts: Each category can have many posts
Independent: No direct relationship with other entities
How It Works Together
-- Get all posts in "Web Development" category
SELECT p.*, c.name as category_name, c.color
FROM posts p
JOIN categories c ON p.category_id = c.id
WHERE c.slug = 'web-development' AND p.status = 'published';

3. Tags Table (Flexible Labeling)
Fields Explained
tags (
  id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  slug TEXT NOT NULL UNIQUE,
  description TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
)

Field Breakdown:
name: Tag display name
Example: "Flutter", "Tutorial", "Beginner"
slug: URL-friendly version
Example: "flutter", "tutorial", "beginner"
Usage: Tag pages like /tag/flutter
Relationships
many:many with posts: Through post_tags junction table
Independent: Can exist without being used
How It Works Together
-- Get all posts tagged with "Flutter"
SELECT p.*, array_agg(t.name) as tags
FROM posts p
JOIN post_tags pt ON p.id = pt.post_id
JOIN tags t ON pt.tag_id = t.id
WHERE t.slug = 'flutter'
GROUP BY p.id;

4. Posts Table (Core Content)
Fields Explained
posts (
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
)

Content Fields:
title: Post headline
slug: URL identifier (e.g., "getting-started-with-jaspr")
excerpt: Short summary for listings and SEO
content_md: Markdown source (what authors write)
content_html: Pre-rendered HTML (performance optimization)
Relationships:
author_id: Links to profiles table
category_id: Links to categories table (optional)
Status Management:
status: Workflow state
'draft': Work in progress, not public
'scheduled': Approved, waiting for publish time
'published': Live and public
'archived': No longer active but preserved
SEO & Marketing:
featured: Highlight important posts
hero_image_url: Featured image URL
hero_image_alt: Accessibility description
meta_title: Custom SEO title
meta_description: SEO description
Analytics:
view_count: Page views (updated by triggers)
like_count: User likes (updated by triggers)
comment_count: Comments (updated by triggers)
reading_time_minutes: Estimated read time
Search:
search_vector: Full-text search index
Weights: Title (A), Excerpt (B), Content (C)
Usage: Fast search across all text content
How Posts Work With Other Tables
Author Information:
-- Get post with author details
SELECT p.*, pr.full_name, pr.avatar_url, pr.bio
FROM posts p
JOIN profiles pr ON p.author_id = pr.id
WHERE p.slug = 'my-post-slug';

Category and Tags:
-- Get post with category and all tags
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

5. Post_Tags Table (Many-to-Many Junction)
Fields Explained
post_tags (
  post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
  tag_id UUID REFERENCES tags(id) ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
)

Why This Table Exists:
Posts can have multiple tags
Tags can be used on multiple posts
Junction table enables many-to-many relationship
How It Works:
-- Add tags to a post
INSERT INTO post_tags (post_id, tag_id) VALUES
  ('post-uuid-1', 'flutter-tag-uuid'),
  ('post-uuid-1', 'tutorial-tag-uuid'),
  ('post-uuid-1', 'beginner-tag-uuid');

-- Remove a tag from a post
DELETE FROM post_tags
WHERE post_id = 'post-uuid-1' AND tag_id = 'flutter-tag-uuid';

6. Comments Table (User Engagement)
Fields Explained
comments (
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
)

User Types:
Authenticated: author_id is set, author_name/email are null
Anonymous: author_id is null, author_name/email are provided
Threading:
parent_id: Enables nested replies
null: Top-level comment
UUID: Reply to another comment
Moderation:
is_approved: Spam protection
Default false: Requires approval for anonymous comments
Authenticated users: Can be auto-approved based on role
How Comments Work
Thread Structure:
Comment 1 (parent_id: null)
├── Reply 1.1 (parent_id: comment-1-id)
│   └── Reply 1.1.1 (parent_id: reply-1.1-id)
└── Reply 1.2 (parent_id: comment-1-id)
Comment 2 (parent_id: null)

Fetch Comments with Replies:
-- Get all comments for a post with author info
SELECT
  c.*,
  pr.full_name,
  pr.avatar_url,
  CASE
    WHEN c.author_id IS NOT NULL THEN pr.full_name
    ELSE c.author_name
  END as display_name
FROM comments c
LEFT JOIN profiles pr ON c.author_id = pr.id
WHERE c.post_id = 'post-uuid' AND c.is_approved = true
ORDER BY c.created_at;

7. Post_Views Table (Analytics)
Fields Explained
post_views (
  id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  post_id UUID REFERENCES posts(id) ON DELETE CASCADE NOT NULL,
  user_id UUID REFERENCES profiles(id) ON DELETE SET NULL,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
)

Tracking Strategy:
user_id: Track authenticated users (can be null)
ip_address: Prevent duplicate views from same IP
user_agent: Detect bots vs real users
How It Works:
-- Record a view (in your Jaspr handler)
INSERT INTO post_views (post_id, user_id, ip_address, user_agent)
VALUES ('post-uuid', 'user-uuid-or-null', '192.168.1.1', 'Mozilla/5.0...')
ON CONFLICT DO NOTHING; -- Prevent duplicates

-- Trigger updates post.view_count automatically

8. Post_Likes Table (User Engagement)
Fields Explained
post_likes (
  post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
  user_id UUID REFERENCES profiles(id) ON DELETE CASCADE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  PRIMARY KEY (post_id, user_id)
)

Constraints:
Composite Primary Key: One like per user per post
Requires Authentication: Only logged-in users can like
How It Works:
-- Like a post
INSERT INTO post_likes (post_id, user_id)
VALUES ('post-uuid', 'user-uuid')
ON CONFLICT DO NOTHING;

-- Unlike a post
DELETE FROM post_likes
WHERE post_id = 'post-uuid' AND user_id = 'user-uuid';

-- Check if user liked a post
SELECT EXISTS(
  SELECT 1 FROM post_likes
  WHERE post_id = 'post-uuid' AND user_id = 'user-uuid'
) as user_liked;

9. Media Table (Asset Management)
Fields Explained
media (
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
)

File Management:
filename: Generated unique name (e.g., "uuid-image.jpg")
original_filename: User's original file name
file_path: Storage path in Supabase Storage
file_size: Bytes for storage tracking
mime_type: File type for validation
Accessibility:
alt_text: Screen reader description
caption: Visible description
Usage in Posts:
![Alt text](https://supabase-storage-url/blog-images/uuid-image.jpg)

How Everything Works Together

1. Publishing Workflow
-- 1. Author creates draft
INSERT INTO posts (title, slug, content_md, author_id, status)
VALUES ('My New Post', 'my-new-post', '# Content here', 'author-uuid', 'draft');

-- 2. Add category and tags
UPDATE posts SET category_id = 'category-uuid' WHERE id = 'post-uuid';
INSERT INTO post_tags (post_id, tag_id) VALUES ('post-uuid', 'tag-uuid');

-- 3. Schedule or publish
UPDATE posts SET
  status = 'published',
  published_at = NOW()
WHERE id = 'post-uuid';

2. Reading Experience
-- Get complete post data for reader
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
    SELECT 1 FROM post_likes
    WHERE post_id = p.id AND user_id = 'current-user-uuid'
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

3. Search Functionality
-- Full-text search across posts
SELECT p.*, pr.full_name as author_name,
       ts_rank(p.search_vector, plainto_tsquery('english', 'flutter tutorial')) as rank
FROM posts p
JOIN profiles pr ON p.author_id = pr.id
WHERE p.search_vector @@ plainto_tsquery('english', 'flutter tutorial')
  AND p.status = 'published'
ORDER BY rank DESC;

4. Analytics Dashboard
-- Get post performance metrics
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

This schema design provides a robust foundation for a modern blogging platform with proper relationships, performance optimization, and extensibility for future features.
