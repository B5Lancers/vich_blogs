Blogging Platform Setup

1. Monorepo Structure
Organize the project as a multi-package (monorepo) repository. For example:
/server – Jaspr (Dart) SSR application code and server entrypoint

/admin – Flutter Web admin dashboard code

/shared – (optional) shared Dart code or models between server and admin

/scripts – CI/CD helper scripts (sitemap generation, RSS, deployment)

Keep code and configs separate: the Jaspr SSR server (running on Cloud Run) and the Flutter Web bundles (served as static assets) live in the same repo but in different folders. This makes CI/CD easier (build both parts in one pipeline).
2. Supabase Backend Setup
a. Create a Supabase Project: Use the Supabase dashboard to create a new project. Enable the database (PostgreSQL), Auth, and Storage. In Auth, configure social logins or email/password as needed for admins.
b. Define Database Tables: In Supabase Studio (Table Editor) or via SQL migrations, create tables like posts, users, tags, etc. For a blog, a minimal posts table might have:
CREATE TABLE public.posts (
  id           uuid DEFAULT uuid_generate_v4() PRIMARY KEY,
  author_id    uuid REFERENCES auth.users(id) ON DELETE SET NULL,
  title        text NOT NULL,
  slug         text UNIQUE NOT NULL,
  content      text NOT NULL,   -- Markdown or HTML body
  excerpt      text,            -- Short summary
  image_url    text,            -- Cover image (Supabase Storage URL)
  published    bool DEFAULT false,
  published_at timestamp with time zone,
  created_at   timestamp with time zone DEFAULT now(),
  updated_at   timestamp with time zone DEFAULT now()
);

Use public.posts in the default schema. By default, tables created in Supabase have Row Level Security (RLS) enabled. If you create a table via SQL, remember to run:

 ALTER TABLE public.posts ENABLE ROW LEVEL SECURITY;

Define other tables as needed (e.g. tags, categories, pivot tables, etc.).

c. RLS and Roles: Configure RLS policies to secure your data. For example:
Public Reader: A policy allowing anyone (or any authenticated user) to select published posts:

 CREATE POLICY select_published ON public.posts
  FOR SELECT USING (published = true);

Admin Access: A role (e.g. admin or use Supabase Auth’s “supabase_owner” role) can insert/update/delete. You could base this on the authenticated user's ID or a custom claim. For example, allow inserts only by authenticated admins:

 CREATE POLICY admins_modify ON public.posts
  FOR ALL USING (auth.role() = 'authenticated' AND auth.uid() IN (
    SELECT id FROM public.users WHERE role = 'admin'
  ));
 (Adjust this logic based on how you mark admin users.)
 In general, use RLS for app-level access control and optionally compose it with Postgres roles/groups.

d. Storage (Images, Media): Use Supabase Storage for uploading images and other assets. Create a bucket (e.g. post_images) and configure its public access settings (typically public-read for front-end serving or use RLS policies on storage tables). The storage has built-in CDN and image transformation (e.g. dynamic resizing). From your admin UI, use the Supabase Dart/Flutter client to upload files to the bucket and store the returned URL in your posts.image_url.
e. Authentication: Use Supabase Auth to manage users. The admin UI (Flutter Web) will prompt admins to log in. You can enforce Supabase’s JWT in your Jaspr server if you make API calls, or simply trust Supabase-authenticated API calls from Flutter. In any case, ensure that write actions (creating/editing posts) require a valid auth token (you can verify on the client or pass through to Supabase policies).
3. Jaspr SSR Shell with Meta Tags (SEO)
a. Jaspr Project: In /server, initialize a Dart project with Jaspr. In pubspec.yaml, include:
dependencies:
  jaspr: ^<latest-version>
  jaspr_flutter_embed: ^<latest-version>   # for embedding Flutter modules
  jaspr_tailwind: ...                      # if using CSS frameworks

In your server code (e.g. bin/main.dart), use Jaspr’s server entrypoint:
import 'package:jaspr/server.dart';
import 'app.dart';  // Your root Jaspr component
import 'jaspr_options.dart';

void main() {
  Jaspr.initializeApp(options: defaultJasprOptions);
  runApp(Document(
    head: [
      // Example: include the compiled Flutter Web script
      script(src: 'main.dart.js', []),
    ],
    body: App(),  // Your top-level Jaspr component
  ));
}

This tells Jaspr to render the App() component into the HTML <body>, and to include the client-side Flutter bundle (main.dart.js) for hydration.
b. Meta Tags and Titles: Use Jaspr’s Document and Document.head() to set HTML metadata. For static/default values, you can pass parameters to Document() (title, lang, meta). For page-specific values, call Document.head() inside your route components. For example, in an article page component:
Document.head(
  title: post.title,
  meta: {
    'description': post.excerpt,
    'og:type': 'article',
    'og:title': post.title,
    'og:description': post.excerpt,
    'og:image': post.imageUrl,
    // ... other meta, JSON-LD etc.
  }
),

This injects <title> and <meta> tags on the server-rendered page for SEO. Jaspr automatically renders these on the server.
c. JSON-LD Structured Data: Add a <script type="application/ld+json"> block for each article. For example, in your article component’s build method:
yield script(
  type: 'application/ld+json',
  [
    text(r'''
      {
        "@context": "https://schema.org",
        "@type": "Article",
        "headline": "''' + post.title + r'''",
        "author": {"@type": "Person", "name": "''' + post.authorName + r'''"},
        "datePublished": "''' + post.publishedAt.toIso8601String() + r'''",
        "image": "''' + post.imageUrl + r'''",
        "articleBody": "''' + post.excerpt + r'''",
        "url": "''' + siteUrl + '/posts/' + post.slug + r'''"
      }
    ''')
  ]
);

This is based on the Article schema example. It helps search engines generate rich snippets (headline, author, date, etc.) from your content.
4. Flutter Web Frontend and Deferred “Islands”
The interactive parts of each page (comments, search box, related posts, etc.) will be Flutter Web widgets that hydrate on the client. Jaspr will render the static shell and content, and then “hydrate” specific components with Flutter. Jaspr’s @client annotation and Flutter embedding makes this easy:
Automatic Hydration: Mark interactive components with @client. For example:

 @client
class CommentsSection extends StatefulComponent { ... }
 A component annotated with @client will be rendered on the server but then “resumed” on the client. This means event handlers and state work without sending the whole app.

Code Splitting (Deferred Loading): Use Dart’s deferred imports to lazy-load large widgets. For example:

 import 'comments.dart' deferred as comments;
...
class ArticlePage extends StatelessComponent {
  Iterable<Component> build(BuildContext context) sync* {
    yield FutureBuilder(
      future: comments.loadLibrary(),  // load on client
      builder: (ctx, snapshot) {
        if (snapshot.hasData) {
          return comments.CommentsSection(postId: post.id);
        } else {
          return placeholderComponent();
        }
      }
    );
  }
}
 This ensures the Comments code is only downloaded when needed, reducing initial JS bundle size. Use FlutterEmbedView.deferred() if using jaspr_flutter_embed (see below) for even finer control.

Jaspr–Flutter Embedding: To include a Flutter Web app/widget inside a Jaspr page, use jaspr_flutter_embed. First, in your Dart code:

 import 'package:jaspr_flutter_embed/jaspr_flutter_embed.dart';
import 'my_flutter_app.dart';
...
class JasprApp extends StatelessComponent {
  Iterable<Component> build(BuildContext context) sync* {
    yield h1([text('Article Title')]);
    yield p([text('Some static content...')]);
    // Embedded Flutter widget:
    yield FlutterEmbedView(
      loader: LoadingWidget(),         // Shown while Flutter loads
      widget: MyFlutterApp(title: 'Demo'),
    );
  }
}
 This wraps the Flutter app (MyFlutterApp) as a component in Jaspr. It will render a <div> placeholder and include Flutter boot scripts behind the scenes. The loader shows until Flutter finishes loading. Use the deferred() constructor of FlutterEmbedView for lazy-loading the Flutter framework.

Conditional Imports for Flutter: Because any code importing flutter must run on the client only, use Dart’s conditional imports or the @Import annotation to separate server vs client code. For example, have a main.dart entrypoint (Jaspr server) that does not import MyFlutterApp, and a separate web/main.dart (Jaspr client) that imports the Flutter web boot scripts and calls runApp().

5. Mediavine Ad Integration
Integrate Mediavine by inserting their script wrapper into your pages. Mediavine provides a JavaScript snippet (often inserted into <head> or just above </body>) that will automatically place ads in your content. For example, in Document.head or just before closing body:

<script id="mvjs">
  /* Your Mediavine script wrapper code (provided by Mediavine) */
</script>

Within your article content, the Mediavine script will detect “blocks” of text (paragraphs, divs) and insert ads accordingly. The ads are lazy-loaded and asynchronous, so they don’t block rendering. If you need precise control, you can also use Mediavine “content hints” (HTML placeholders like <div class="content_hint"></div>) to fix placement. In practice, include the script wrapper once (in Document.head() or a template) and ensure your article HTML has enough paragraphs/divs for Mediavine’s logic to place ads.
6. Docker & Cloud Run Deployment
a. Dockerfile: Use Jaspr’s recommended Dockerfile (from Jaspr docs) to build a container image. Example (in /server/Dockerfile):

# Build stage with Dart (and Flutter if needed)

FROM dart:stable AS build
RUN dart pub global activate jaspr_cli
WORKDIR /app
COPY . .
RUN dart pub get
RUN dart pub global run jaspr_cli:jaspr build --verbose

# Runtime image

FROM scratch

# Copy Dart runtime libs

COPY --from=build /runtime/ /

# Copy the built site

COPY --from=build /app/build/jaspr/ /app/
WORKDIR /app

EXPOSE 8080
CMD ["./app"]  # Runs the compiled server executable

This uses Dart’s official image to compile the app and then copies only the built artifacts into a minimal image. The CMD ["./app"] runs the server on port 8080 (Cloud Run expects HTTP on $PORT).
b. Flutter Support: If you use Flutter embedding or packages requiring Flutter, start the build stage from a Flutter image:
FROM ghcr.io/cirruslabs/flutter:stable AS build
...

Then copy the Dart runtime as shown in Jaspr docs. This ensures Flutter SDK is available to compile the Flutter parts.
c. Cloud Run: Push the container to a registry (e.g. Google Container Registry or Artifact Registry). Then deploy on Cloud Run. In Google Cloud Run settings, set minimum instances = 1 to avoid cold starts (so the app is always ready). Use HTTP/HTTPS settings as normal. Cloud Run will then serve your Jaspr SSR application.
7. Static Assets & CDN
Place all static assets (Flutter Web JS bundles, images, sitemap, RSS, etc.) on a CDN. Options include:
Cloudflare CDN – Upload the /build/flutter/ web bundles and generated sitemap.xml/rss.xml to Cloudflare (e.g. via Workers KV, Pages, or Cloudflare R2) so they’re served globally with low latency. Cloudflare’s CDN automatically caches and serves static content.

AWS S3 + CloudFront – Alternatively, store assets in an S3 bucket and use CloudFront distribution. This is a common static hosting setup (see AWS docs on serving a static site via CloudFront).

In CI/CD (below) after building the Flutter web project (e.g. running flutter build web), sync the output to your static host. Then configure Cloudflare/CloudFront with your custom domain or caching rules.
Also include an up-to-date sitemap.xml and rss.xml for SEO. You can generate these during CI by listing all blog URLs (e.g. fetch from Supabase). For example, use a Dart or JS script to write an XML file like:
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url><loc>https://example.com/</loc><lastmod>2025-06-10</lastmod></url>
  <url><loc>https://example.com/posts/first-post</loc><lastmod>2025-06-01</lastmod></url>
  <!-- more URLs -->
</urlset>

And an RSS feed in XML for your blog. Push these to the CDN too. Google’s Search Console can then crawl them.
8. Admin Dashboard (Flutter Web)
Create a Flutter Web app (in /admin) for content management. It should allow admins to log in (using Supabase Auth), and then create/edit/schedule/publish posts. Key features:
Login Screen: Use Supabase’s Dart/Flutter libraries (supabase_flutter) to authenticate.

Post Editor: A form for title, slug, content (e.g. Markdown editor or rich text), excerpt, image upload, publish date. Implement autosave (e.g. on every keystroke or at intervals, save draft to Supabase via API or realtime DB). For image upload, use Supabase Storage API to upload and return a URL for image_url.

Scheduling: Allow setting published_at. In your data model, use this to auto-publish posts in the future. You could have an edge function or Cloud Run job that flips published to true when time comes.

Media Library: Optionally integrate a media upload/viewer. You can directly list files from the Supabase bucket.

Routes & State Management: Use Flutter state management (Provider, Riverpod, etc.) and Flutter Web router for navigation (Dashboard, list of posts, editor, settings).

Communicate with Supabase using the supabase_flutter client: fetch tables, insert/update rows, apply RLS automatically. This decouples from Jaspr — it’s purely a client-side Flutter app. You’ll deploy this separately, or as static assets on the CDN (since it’s purely client-side once built).
9. Deferred Interactive Widgets (“Islands”)
For the public site pages, only certain parts need heavy interactivity (comments widget, search box, related-posts carousel, etc.). With Jaspr, use islands architecture: each interactive widget is a separate Flutter component hydrated on the client. For example:
In the article page component, include a placeholder <div> (via Jaspr) where Comments will go, and annotate the CommentsSection with @client so Jaspr hydrates it.

Similarly, a related-posts widget can be its own Flutter component loaded with @client, deferred until after initial render.

By moving the @client as deep in the tree as possible, you minimize the client-side JS shipped. For instance, wrap only the comment list in a @client component, not the whole article. Use Dart deferred imports (as above) or @client for each island. This way, each island loads only when needed, and the rest of the page stays lightweight.
10. CI/CD with GitHub Actions
Automate builds and deploys in GitHub Actions:
Checkout & Setup: Use actions/checkout, then set up Dart and Flutter SDKs (dart-lang/setup-dart@v1.3 and subosito/flutter-action@v2) as needed. Authenticate to Google Cloud with google-github-actions/auth@v2 (using a service account key or Workload Identity).

Build Flutter Web: In the admin or any Flutter web part, run flutter build web (for the client-side modules). The output (e.g. in build/web/) contains JS/CSS files.

Sync Static Assets: Upload the Flutter web build to your CDN. For Cloudflare, you might use wrangler or API scripts; for S3, use aws s3 sync with AWS CLI. Also update sitemap.xml and rss.xml (you can run a Dart or Node script to fetch post slugs from Supabase and write these files), then push them to the bucket/CDN.

Build Docker & Push: In the server directory, run docker build -t $IMAGE_NAME . using your Dockerfile. Tag the image (e.g. to gcr.io/$PROJECT/$IMAGE_NAME:$GITHUB_SHA) and push to your container registry.

Deploy to Cloud Run: Use google-github-actions/deploy-cloudrun@v2 to deploy the new image:

- uses: google-github-actions/auth@v2
  with:
    credentials_json: '${{ secrets.GCP_SA_KEY }}'
- uses: google-github-actions/deploy-cloudrun@v2
  with:
    image: 'gcr.io/$PROJECT/$IMAGE_NAME'
    service: 'my-blog-service'
    region: 'us-central1'
    project_id: '$PROJECT'
 This redeploys the server; Cloud Run will route traffic to it. (If using a custom domain, ensure Cloud Run is set up for that.)

Each push to main can trigger this workflow. For preview environments, you could deploy to a different Cloud Run service.
11. Monitoring & SEO Testing
Error Logging & Monitoring: Integrate Sentry into your Flutter/Web apps for errors and performance. The Sentry Flutter SDK “enables automatic reporting of errors and performance data”. In your Flutter and Dart code, initialize Sentry with your DSN so that any uncaught exceptions or performance traces are sent to Sentry. This covers both the admin Flutter app and any client-side Flutter islands.

Analytics: Add Google Analytics (GA4) or another analytics tool to track page views and user behavior. For Flutter Web, you can use flutter_ga or JS interop to fire GA events.

Performance (Core Web Vitals): Use Google PageSpeed Insights or Lighthouse to test your site’s load/performance scores. Enabling caching (CDN), compressing assets, and lazy-loading widgets (as above) will improve scores.

SEO Testing: Use Google Search Console (GSC) to submit your sitemap and monitor indexing. “Setting up a Search Console account helps you monitor and optimize how your website performs on Google Search”. In GSC you can inspect pages, check for errors (e.g. mobile usability), and see which queries lead to your site. Also use tools like Google’s Rich Results Test and PageSpeed Insights to verify structured data and performance.

By following these steps – from database design with RLS, through a Jaspr SSR shell, Flutter Web islands, and robust CI/CD – you’ll have a fast, SEO-friendly blog platform. Each paragraph above contains references for key tools and practices (e.g. Jaspr’s meta tags, hydration with @client, and Mediavine’s ad wrapper).
Sources: Jaspr documentation and guides; Supabase docs; Cloud Run and GitHub Actions docs; Mediavine help center; Google Search Central (SEO); Sentry Flutter docs.

Response 2:

Here’s a comprehensive, ordered roadmap you can follow to build your Jaspr / Flutter Web blogging platform with the SSR + CSR hybrid approach, Supabase backend, Mediavine ads, and full CI/CD. Each major phase is broken into concrete steps.

Phase 0: Prep & Planning
Repository & Monorepo Structure

Create a Git repo (e.g. on GitHub).

Organize as a monorepo with two top-level packages:

 /packages
  ├─ blog_frontend/    # Flutter Web “reader” + deferred islands
  └─ blog_admin/       # Flutter Web admin/editor UI
/server
  └─ jaspr_ssr/        # Jaspr SSR shell (Dart package)
/infra
  ├─ ci/               # GitHub Actions workflows
  └─ k8s/              # (optional) Kubernetes manifests

Tech Stack Confirmation

Rendering: Jaspr SSR for shell + Flutter Web CSR hydration.

Backend & Auth: Supabase (Postgres + Auth + Storage + Edge Functions).

Ads: Mediavine script injected client-side.

Hosting: Cloud Run (SSR) + CDN (Flutter assets, sitemap, RSS).

CI/CD: GitHub Actions to build, deploy, and update sitemap/RSS.

Account Setup

Create/verify accounts for:

GitHub

Supabase project

Google Cloud (or Fly.io)

Cloudflare (or S3 + CloudFront) for CDN

Sentry / Rollbar (error monitoring)

Google Analytics & Search Console

Phase 1: Backend & CMS
Supabase Project

Spin up a new Supabase project.

Enable Auth (email/password) and configure social logins if desired.

Database Schema

Create tables:

 -- users (supabase provides)
-- roles (admin, editor, author)
-- posts (
    id           uuid primary key default uuid_generate_v4(),
    author_id    uuid references auth.users(id),
    title        text not null,
    slug         text unique not null,
    content_md   text not null,
    status       text check (status in ('draft','scheduled','published')) not null,
    published_at timestamp,
    hero_image   text,           -- Supabase Storage URL
    inserted_at  timestamp default now(),
    updated_at   timestamp default now()
);
-- tags (id, name)
-- post_tags (post_id, tag_id)

RLS & Policies

users can only read published posts.

authors/editors can insert/update their own drafts.

admins have full access.

Use Supabase UI to define RLS policies.

Storage Bucket

Create a blog-images bucket.

Set public read access.

Configure retention / lifecycle as needed.

Edge Functions

(Optional) Write a small function to append new posts to your sitemap & RSS without rebuilding the frontend.

Phase 2: Jaspr SSR Shell
Bootstrap Jaspr Project

 dart create -t server-simple server/jaspr_ssr
cd server/jaspr_ssr
dart pub add jaspr

SSR Handler

Implement a route for GET /posts/:slug that:

Fetches minimal post metadata from Supabase (title, description, hero_image, published_at, author).

Renders an HTML page with:

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>{{title}}</title>
  <meta name="description" content="{{excerpt}}">
  <meta property="og:title" content="{{title}}">
  <meta property="og:description" content="{{excerpt}}">
  <meta property="og:image" content="{{hero_image}}">
  <script type="application/ld+json">
    { /* JSON-LD Article schema */ }
  </script>
  <!-- Mediavine placeholder -->
  <div id="mediavine-ad-1"></div>
</head>
<body>
  <div id="flutter-root"><!-- Flutter app mounts here --></div>
  <script src="/assets/loader.js"></script> <!-- hydration loader -->
</body>
</html>

Containerization

Add a Dockerfile:

 FROM dart:stable AS build
WORKDIR /app
COPY pubspec.* ./
RUN dart pub get
COPY . .
RUN dart compile exe bin/server.dart -o server

FROM gcr.io/distroless/cc
COPY --from=build /app/server /server
EXPOSE 8080
CMD ["/server"]

Phase 3: Flutter Web Frontend (“Reader”)
Create Flutter Project

 flutter create packages/blog_frontend
cd packages/blog_frontend

Folder Layout (Clean Architecture)

 lib/
├─ main.dart            # entrypoint
├─ app/                 # routing & DI
├─ features/
│   └─ article/
│       ├─ presentation/
│       │   ├─ article_screen.dart
│       │   ├─ article_bloc.dart
│       ├─ domain/
│       │   └─ article_entity.dart
│       └─ data/
│           └─ article_repository.dart
└─ shared/
    ├─ widgets/         # code blocks, images, markdown renderer
    └─ services/        # supabase_client.dart

Deferred Islands

In main.dart, set up:

 import 'article_screen.dart' deferred as article;
// Later, when route matches:
await article.loadLibrary();
return article.ArticleScreen(...);

Repeat for comments, related posts, search.

Supabase SDK

Add & configure supabase_flutter.

Initialize in main.dart with your project URL & anon key.

Markdown Rendering

Use flutter_markdown to render content_md.

Customize styles for code snippets, images.

Mediavine Injection

After your article widget builds, call:

 import 'dart:js' as js;
js.context.callMethod('eval', ["window.Mediavine.go('ad-1');"]);

Phase 4: Admin / Editor UI
Create Flutter Web Project

 flutter create packages/blog_admin

Auth Flow

Reuse supabase_flutter for login.

Guard pages by role (author/editor/admin).

Editor Screen

Build a Markdown editor (e.g. flutter_markdown + TextField).

Implement autosave: debounce user input → save draft to posts table with status = 'draft'.

Scheduling UI: date-picker for published_at + setting status = 'scheduled'.

Media Upload

Integrate Supabase Storage picker for images.

Return URL and insert into Markdown content.

Tag Management

CRUD UI for tags.

Multi-select in post editor.

Phase 5: CI/CD & Hosting
GitHub Actions (.github/workflows/ci.yml)

 name: CI

on:
  push:
    branches: [ main ]

jobs:
  build-ssr:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build & Push SSR Docker
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ secrets.GCLOUD_REGISTRY }}/jaspr-ssr:latest

  build-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
      - name: Build Web
        run: flutter build web --release --target=lib/main_web.dart
      - name: Deploy to CDN
        run: |
          aws s3 sync build/web s3://your-cdn-bucket --delete
          aws cloudfront create-invalidation --distribution-id YOUR_ID --paths "/*"

  update-sitemap:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Trigger Edge Function
        run: curl -X POST https://YOUR_EDGE_FN/endpoints/update-sitemap

Deploy SSR Container

 gcloud run deploy blog-ssr \
  --image gcr.io/your-project/jaspr-ssr:latest \
  --min-instances 1 \
  --region us-central1 \
  --allow-unauthenticated

CDN Configuration

On Cloudflare/S3+CloudFront, point /* to your Flutter assets.

Route /posts/* to your SSR service (via a worker or URL routing).

Phase 6: Monitoring, Testing & Rollout
Local Dev

Run Jaspr shell locally (dart run bin/server.dart).

Run Flutter Web in HMR (flutter run -d chrome --web-port=5000).

Proxy routing so /posts/:slug hits your local SSR.

Testing

Unit tests for Blocs & Repos.

Integration tests with Cypress (optional) against deployed preview.

Monitoring

Integrate web_vitals via JS interop.

Configure Sentry for both Dart (SSR) and JS (Flutter Web).

Verify Google Search Console indexing & RSS feed validity.

Phased Launch

Alpha: SSR shell + core “reader” module only.

Beta: Add deferred comments & related-posts islands.

GA: Full admin UI, scheduling, tags, and Mediavine ads live.

🎉 You’re all set to build, deploy, and scale your own SEO-friendly, ad-monetized Flutter-powered blog! Let me know if you’d like deeper dives on any individual step—folder structures, CI tweaks, Edge Function code, or anything else.
