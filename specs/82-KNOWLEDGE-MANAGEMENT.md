# 82 - Knowledge Management Service Specification

## 1. Domain Overview

| Attribute | Value |
|---|---|
| **Bounded Context** | Knowledge Management |
| **Service Name** | knowledge-service |
| **Database** | knowledge_db |
| **HTTP Port** | 8114 |
| **gRPC Port** | 9114 |

The Knowledge Management module provides a centralized knowledge base for creating, reviewing, publishing, and maintaining articles and content. It supports versioned articles with multi-language translations, categorization, tagging, rating, and usage analytics. The system includes a full draft-review-publish workflow engine and integrates with customer service, digital assistant, and HR help desk modules to deliver contextual knowledge to agents, customers, and employees.

---

## 2. Database Schema

```sql
CREATE TABLE knowledge_articles (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    article_number TEXT NOT NULL UNIQUE,
    title TEXT NOT NULL,
    summary TEXT NOT NULL,
    body TEXT NOT NULL,
    format TEXT NOT NULL DEFAULT 'HTML',
    locale TEXT NOT NULL DEFAULT 'en',
    category_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'DRAFT',
    content_version INTEGER NOT NULL DEFAULT 1,
    latest_version INTEGER NOT NULL DEFAULT 1,
    parent_article_id TEXT,
    owner_id TEXT NOT NULL,
    reviewer_id TEXT,
    published_at TEXT,
    expires_at TEXT,
    review_deadline TEXT,
    effective_start_date TEXT,
    effective_end_date TEXT,
    view_count INTEGER NOT NULL DEFAULT 0,
    helpful_count INTEGER NOT NULL DEFAULT 0,
    not_helpful_count INTEGER NOT NULL DEFAULT 0,
    average_rating INTEGER,
    is_featured INTEGER NOT NULL DEFAULT 0,
    is_internal INTEGER NOT NULL DEFAULT 0,
    seo_keywords TEXT,
    search_keywords TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_knowledge_articles_tenant ON knowledge_articles(tenant_id);
CREATE INDEX idx_knowledge_articles_status ON knowledge_articles(tenant_id, status);
CREATE INDEX idx_knowledge_articles_category ON knowledge_articles(tenant_id, category_id);
CREATE INDEX idx_knowledge_articles_owner ON knowledge_articles(tenant_id, owner_id);
CREATE INDEX idx_knowledge_articles_locale ON knowledge_articles(tenant_id, locale);
CREATE INDEX idx_knowledge_articles_search ON knowledge_articles(tenant_id, search_keywords);
CREATE INDEX idx_knowledge_articles_published ON knowledge_articles(tenant_id, published_at);

CREATE TABLE article_categories (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    category_code TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    parent_category_id TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,
    article_count INTEGER NOT NULL DEFAULT 0,
    visibility TEXT NOT NULL DEFAULT 'PUBLIC',
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_article_categories_tenant ON article_categories(tenant_id);
CREATE INDEX idx_article_categories_parent ON article_categories(parent_category_id);

CREATE TABLE article_tags (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    article_id TEXT NOT NULL,
    tag TEXT NOT NULL,
    tag_type TEXT NOT NULL DEFAULT 'USER',
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_article_tags_article ON article_tags(tenant_id, article_id);
CREATE INDEX idx_article_tags_tag ON article_tags(tenant_id, tag);
CREATE INDEX idx_article_tags_tenant_tag ON article_tags(tenant_id, tag, tag_type);

CREATE TABLE article_translations (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    source_article_id TEXT NOT NULL,
    locale TEXT NOT NULL,
    translated_title TEXT NOT NULL,
    translated_summary TEXT NOT NULL,
    translated_body TEXT NOT NULL,
    translation_status TEXT NOT NULL DEFAULT 'PENDING',
    translator_id TEXT,
    reviewer_id TEXT,
    translated_at TEXT,
    reviewed_at TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_article_translations_source ON article_translations(tenant_id, source_article_id);
CREATE INDEX idx_article_translations_locale ON article_translations(tenant_id, locale);
CREATE UNIQUE INDEX idx_article_translations_unique ON article_translations(tenant_id, source_article_id, locale);

CREATE TABLE article_ratings (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    article_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    user_type TEXT NOT NULL DEFAULT 'CUSTOMER',
    rating INTEGER NOT NULL,
    is_helpful INTEGER,
    comments TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_article_ratings_article ON article_ratings(tenant_id, article_id);
CREATE INDEX idx_article_ratings_user ON article_ratings(tenant_id, user_id, user_type);

CREATE TABLE article_usage_analytics (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    article_id TEXT NOT NULL,
    event_type TEXT NOT NULL,
    user_id TEXT,
    user_type TEXT,
    session_id TEXT,
    search_query TEXT,
    search_position INTEGER,
    dwell_time_seconds INTEGER,
    referrer_source TEXT,
    channel TEXT,
    locale TEXT,
    ip_address TEXT,
    user_agent TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_article_analytics_article ON article_usage_analytics(tenant_id, article_id);
CREATE INDEX idx_article_analytics_event ON article_usage_analytics(tenant_id, event_type);
CREATE INDEX idx_article_analytics_created ON article_usage_analytics(tenant_id, created_at);

CREATE TABLE knowledge_workflows (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    article_id TEXT NOT NULL,
    article_version INTEGER NOT NULL,
    workflow_type TEXT NOT NULL,
    from_status TEXT NOT NULL,
    to_status TEXT NOT NULL,
    submitted_by TEXT NOT NULL,
    assigned_reviewer_id TEXT,
    review_comments TEXT,
    priority TEXT NOT NULL DEFAULT 'NORMAL',
    due_at TEXT,
    completed_at TEXT,
    status TEXT NOT NULL DEFAULT 'PENDING',
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_knowledge_workflows_article ON knowledge_workflows(tenant_id, article_id);
CREATE INDEX idx_knowledge_workflows_reviewer ON knowledge_workflows(tenant_id, assigned_reviewer_id, status);
CREATE INDEX idx_knowledge_workflows_status ON knowledge_workflows(tenant_id, status);

CREATE TABLE related_articles (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    article_id TEXT NOT NULL,
    related_article_id TEXT NOT NULL,
    relationship_type TEXT NOT NULL DEFAULT 'SEE_ALSO',
    relevance_score INTEGER,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_related_articles_article ON related_articles(tenant_id, article_id);
CREATE UNIQUE INDEX idx_related_articles_pair ON related_articles(tenant_id, article_id, related_article_id);

CREATE TABLE article_attachments (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    article_id TEXT NOT NULL,
    file_name TEXT NOT NULL,
    file_type TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    storage_path TEXT NOT NULL,
    description TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_article_attachments_article ON article_attachments(tenant_id, article_id);

CREATE TABLE search_index_metadata (
    id TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    article_id TEXT NOT NULL,
    article_version INTEGER NOT NULL,
    indexed_title TEXT,
    indexed_body TEXT,
    indexed_keywords TEXT,
    indexed_tags TEXT,
    relevance_boost INTEGER NOT NULL DEFAULT 0,
    last_indexed_at TEXT NOT NULL,
    index_status TEXT NOT NULL DEFAULT 'CURRENT',
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    created_by TEXT NOT NULL,
    updated_by TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_search_index_tenant ON search_index_metadata(tenant_id);
CREATE INDEX idx_search_index_article ON search_index_metadata(tenant_id, article_id);
CREATE INDEX idx_search_index_status ON search_index_metadata(tenant_id, index_status);
```

---

## 3. REST API Endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/articles` | Create a new knowledge article (draft) |
| GET | `/api/v1/articles/{articleId}` | Get article details by ID |
| GET | `/api/v1/articles` | List/search articles with filters |
| PUT | `/api/v1/articles/{articleId}` | Update article content |
| DELETE | `/api/v1/articles/{articleId}` | Soft-delete an article |
| POST | `/api/v1/articles/{articleId}/submit-review` | Submit article for review |
| POST | `/api/v1/articles/{articleId}/approve` | Approve article in review |
| POST | `/api/v1/articles/{articleId}/reject` | Reject article in review |
| POST | `/api/v1/articles/{articleId}/publish` | Publish approved article |
| POST | `/api/v1/articles/{articleId}/unpublish` | Unpublish a live article |
| POST | `/api/v1/articles/{articleId}/archive` | Archive an article |
| GET | `/api/v1/articles/{articleId}/versions` | Get article version history |
| GET | `/api/v1/articles/{articleId}/versions/{version}` | Get specific article version |
| POST | `/api/v1/articles/{articleId}/rate` | Rate an article |
| GET | `/api/v1/articles/{articleId}/ratings` | Get ratings for article |
| POST | `/api/v1/articles/{articleId}/translations` | Create/update article translation |
| GET | `/api/v1/articles/{articleId}/translations` | List translations for article |
| GET | `/api/v1/articles/{articleId}/translations/{locale}` | Get specific translation |
| POST | `/api/v1/articles/{articleId}/attachments` | Upload attachment to article |
| GET | `/api/v1/articles/{articleId}/attachments` | List attachments for article |
| DELETE | `/api/v1/articles/{articleId}/attachments/{attachmentId}` | Remove attachment |
| POST | `/api/v1/articles/{articleId}/related` | Link related article |
| GET | `/api/v1/articles/{articleId}/related` | Get related articles |
| DELETE | `/api/v1/articles/{articleId}/related/{relatedId}` | Remove related article link |
| POST | `/api/v1/articles/{articleId}/tags` | Add tags to article |
| GET | `/api/v1/articles/{articleId}/tags` | Get tags for article |
| DELETE | `/api/v1/articles/{articleId}/tags/{tagId}` | Remove tag from article |
| GET | `/api/v1/search` | Full-text search across knowledge base |
| POST | `/api/v1/search` | Advanced search with filters |
| GET | `/api/v1/categories` | List article categories |
| POST | `/api/v1/categories` | Create article category |
| GET | `/api/v1/categories/{categoryId}` | Get category details |
| PUT | `/api/v1/categories/{categoryId}` | Update category |
| GET | `/api/v1/workflows` | List workflow tasks (review queue) |
| GET | `/api/v1/workflows/{workflowId}` | Get workflow task details |
| POST | `/api/v1/workflows/{workflowId}/complete` | Complete a workflow task |
| GET | `/api/v1/analytics/usage` | Get article usage analytics |
| GET | `/api/v1/analytics/popular` | Get most-viewed articles |
| GET | `/api/v1/analytics/ratings-summary` | Get ratings summary |
| GET | `/api/v1/analytics/search-effectiveness` | Get search effectiveness metrics |
| GET | `/api/v1/analytics/gaps` | Identify knowledge gap areas |
| POST | `/api/v1/articles/{articleId}/view` | Record article view event |
| GET | `/api/v1/tags` | List all unique tags across tenant |
| GET | `/api/v1/articles/{articleId}/export` | Export article as PDF/HTML |

---

## 4. Business Rules

1. Every article MUST be created in DRAFT status; articles cannot be created directly in PUBLISHED status.
2. An article MUST go through the full draft-review-publish workflow before being visible to external users; the workflow MUST include at least one reviewer approval step.
3. Article versions MUST be tracked immutably; when an article is updated after publication, a new content version MUST be created while the previous version remains accessible.
4. The `version` column on knowledge_articles tracks the record optimistic lock version, while `content_version` tracks content versions; both MUST be maintained independently.
5. A reviewer MUST NOT be the same person as the article owner; self-review MUST be rejected by the system.
6. Published articles with an expires_at date MUST be automatically transitioned to EXPIRED status by a scheduled job when the current date passes the expiration date.
7. Article ratings MUST be an integer between 1 and 5; each user (identified by user_id and user_type) MAY submit only one rating per article.
8. Translations MUST reference a source article and locale; each source article MAY have at most one translation per locale.
9. Search results MUST be ranked by a combination of full-text relevance, rating score, view count, and recency of last update; administrators SHOULD be able to boost specific articles.
10. Knowledge gap analysis SHOULD identify search queries with zero or low-quality results and flag them for article creation.
11. Article categories MUST support a hierarchical structure with unlimited depth; circular references in parent_category_id MUST be prevented.
12. Related article links MUST be bidirectional; creating a relationship from A to B SHOULD automatically create the inverse relationship.
13. Usage analytics events MUST be recorded asynchronously and MUST NOT block article retrieval operations.
14. Attachments MUST have a maximum file size of 50 MB per file and 200 MB total per article; supported file types SHOULD be configurable per tenant.
15. The search index MUST be updated within 5 seconds of any article publication or unpublish event.
16. All timestamps MUST be stored and transmitted in ISO 8601 format with UTC timezone.
17. Tenant isolation MUST be enforced on every query; no cross-tenant data access is permitted.
18. Articles marked as is_internal = 1 MUST NOT be returned in searches initiated by external users or customer-facing channels.

---

## 5. gRPC Service Definition

```protobuf
syntax = "proto3";

package knowledge.v1;

option go_package = "github.com/fusion/protos/knowledge/v1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

service KnowledgeManagement {
    // Article CRUD
    rpc CreateArticle(CreateArticleRequest) returns (ArticleResponse);
    rpc GetArticle(GetArticleRequest) returns (ArticleResponse);
    rpc ListArticles(ListArticlesRequest) returns (ListArticlesResponse);
    rpc UpdateArticle(UpdateArticleRequest) returns (ArticleResponse);
    rpc DeleteArticle(DeleteArticleRequest) returns (google.protobuf.Empty);

    // Workflow
    rpc SubmitForReview(SubmitForReviewRequest) returns (ArticleResponse);
    rpc ApproveArticle(ApproveArticleRequest) returns (ArticleResponse);
    rpc RejectArticle(RejectArticleRequest) returns (ArticleResponse);
    rpc PublishArticle(PublishArticleRequest) returns (ArticleResponse);
    rpc UnpublishArticle(UnpublishArticleRequest) returns (ArticleResponse);
    rpc ArchiveArticle(ArchiveArticleRequest) returns (ArticleResponse);

    // Versions
    rpc GetArticleVersions(GetArticleVersionsRequest) returns (ArticleVersionsResponse);
    rpc GetArticleVersion(GetArticleVersionRequest) returns (ArticleResponse);

    // Search
    rpc SearchArticles(SearchArticlesRequest) returns (SearchArticlesResponse);
    rpc AdvancedSearch(AdvancedSearchRequest) returns (SearchArticlesResponse);

    // Ratings
    rpc RateArticle(RateArticleRequest) returns (RateArticleResponse);
    rpc GetArticleRatings(GetArticleRatingsRequest) returns (ArticleRatingsResponse);

    // Translations
    rpc CreateTranslation(CreateTranslationRequest) returns (TranslationResponse);
    rpc GetTranslation(GetTranslationRequest) returns (TranslationResponse);
    rpc ListTranslations(ListTranslationsRequest) returns (ListTranslationsResponse);
    rpc UpdateTranslation(UpdateTranslationRequest) returns (TranslationResponse);

    // Related Articles
    rpc AddRelatedArticle(AddRelatedArticleRequest) returns (RelatedArticleResponse);
    rpc ListRelatedArticles(ListRelatedArticlesRequest) returns (ListRelatedArticlesResponse);
    rpc RemoveRelatedArticle(RemoveRelatedArticleRequest) returns (google.protobuf.Empty);

    // Categories
    rpc CreateCategory(CreateCategoryRequest) returns (CategoryResponse);
    rpc ListCategories(ListCategoriesRequest) returns (ListCategoriesResponse);
    rpc GetCategory(GetCategoryRequest) returns (CategoryResponse);
    rpc UpdateCategory(UpdateCategoryRequest) returns (CategoryResponse);

    // Workflows (Review Queue)
    rpc ListWorkflowTasks(ListWorkflowTasksRequest) returns (ListWorkflowTasksResponse);
    rpc GetWorkflowTask(GetWorkflowTaskRequest) returns (WorkflowTaskResponse);
    rpc CompleteWorkflowTask(CompleteWorkflowTaskRequest) returns (WorkflowTaskResponse);

    // Analytics
    rpc GetUsageAnalytics(GetUsageAnalyticsRequest) returns (UsageAnalyticsResponse);
    rpc GetPopularArticles(GetPopularArticlesRequest) returns (PopularArticlesResponse);
    rpc RecordArticleView(RecordArticleViewRequest) returns (google.protobuf.Empty);

    // Attachments
    rpc AddAttachment(AddAttachmentRequest) returns (AttachmentResponse);
    rpc ListAttachments(ListAttachmentsRequest) returns (ListAttachmentsResponse);
    rpc RemoveAttachment(RemoveAttachmentRequest) returns (google.protobuf.Empty);

    // Tags
    rpc AddTags(AddTagsRequest) returns (TagsResponse);
    rpc ListTags(ListTagsRequest) returns (TagsListResponse);
    rpc RemoveTag(RemoveTagRequest) returns (google.protobuf.Empty);
}

message ArticleMessage {
    string id = 1;
    string tenant_id = 2;
    string article_number = 3;
    string title = 4;
    string summary = 5;
    string body = 6;
    string format = 7;
    string locale = 8;
    string category_id = 9;
    string status = 10;
    int32 content_version = 11;
    int32 latest_version = 12;
    string parent_article_id = 13;
    string owner_id = 14;
    string reviewer_id = 15;
    google.protobuf.Timestamp published_at = 16;
    google.protobuf.Timestamp expires_at = 17;
    google.protobuf.Timestamp review_deadline = 18;
    int32 view_count = 19;
    int32 helpful_count = 20;
    int32 not_helpful_count = 21;
    int32 average_rating = 22;
    bool is_featured = 23;
    bool is_internal = 24;
    string search_keywords = 25;
    bool is_active = 26;
    google.protobuf.Timestamp created_at = 27;
    google.protobuf.Timestamp updated_at = 28;
    string created_by = 29;
    string updated_by = 30;
    int32 version = 31;
}

message CreateArticleRequest {
    string tenant_id = 1;
    string title = 2;
    string summary = 3;
    string body = 4;
    string format = 5;
    string locale = 6;
    string category_id = 7;
    string owner_id = 8;
    string search_keywords = 9;
    bool is_internal = 10;
    google.protobuf.Timestamp expires_at = 11;
    repeated string tags = 12;
}

message GetArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
}

message ListArticlesRequest {
    string tenant_id = 1;
    string status = 2;
    string category_id = 3;
    string owner_id = 4;
    string locale = 5;
    int32 page_size = 6;
    string page_token = 7;
}

message ListArticlesResponse {
    repeated ArticleMessage articles = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
    string title = 3;
    string summary = 4;
    string body = 5;
    string category_id = 6;
    string search_keywords = 7;
    google.protobuf.Timestamp expires_at = 8;
    int32 version = 9;
}

message DeleteArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
}

message SubmitForReviewRequest {
    string tenant_id = 1;
    string article_id = 2;
    string assigned_reviewer_id = 3;
    string priority = 4;
}

message ApproveArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
    string review_comments = 3;
}

message RejectArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
    string review_comments = 3;
}

message PublishArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
    google.protobuf.Timestamp effective_start_date = 3;
    google.protobuf.Timestamp effective_end_date = 4;
}

message UnpublishArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
    string reason = 3;
}

message ArchiveArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
    string reason = 3;
}

message ArticleResponse {
    ArticleMessage article = 1;
}

message GetArticleVersionsRequest {
    string tenant_id = 1;
    string article_id = 2;
}

message ArticleVersionsResponse {
    repeated ArticleVersionSummary versions = 1;
}

message ArticleVersionSummary {
    int32 content_version = 1;
    string status = 2;
    string title = 3;
    google.protobuf.Timestamp created_at = 4;
    string created_by = 5;
}

message GetArticleVersionRequest {
    string tenant_id = 1;
    string article_id = 2;
    int32 content_version = 3;
}

message SearchArticlesRequest {
    string tenant_id = 1;
    string query = 2;
    string category_id = 3;
    string locale = 4;
    bool include_internal = 5;
    int32 max_results = 6;
}

message AdvancedSearchRequest {
    string tenant_id = 1;
    string query = 2;
    repeated string category_ids = 3;
    repeated string tags = 4;
    string locale = 5;
    string date_from = 6;
    string date_to = 7;
    string sort_by = 8;
    string sort_order = 9;
    int32 page_size = 10;
    string page_token = 11;
}

message SearchArticlesResponse {
    repeated ArticleSearchResult results = 1;
    int32 total_count = 2;
    string next_page_token = 3;
}

message ArticleSearchResult {
    string article_id = 1;
    string title = 2;
    string summary = 3;
    string category_name = 4;
    int32 relevance_score = 5;
    int32 average_rating = 6;
    int32 view_count = 7;
    google.protobuf.Timestamp published_at = 8;
    string locale = 9;
    repeated string matched_highlights = 10;
}

message RateArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
    string user_id = 3;
    string user_type = 4;
    int32 rating = 5;
    bool is_helpful = 6;
    string comments = 7;
}

message RateArticleResponse {
    string rating_id = 1;
    int32 average_rating = 2;
    int32 total_ratings = 3;
}

message GetArticleRatingsRequest {
    string tenant_id = 1;
    string article_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ArticleRatingsResponse {
    repeated RatingMessage ratings = 1;
    double average_rating = 2;
    int32 total_count = 3;
    string next_page_token = 4;
}

message RatingMessage {
    string id = 1;
    string user_id = 2;
    int32 rating = 3;
    bool is_helpful = 4;
    string comments = 5;
    google.protobuf.Timestamp created_at = 6;
}

message CreateTranslationRequest {
    string tenant_id = 1;
    string source_article_id = 2;
    string locale = 3;
    string translated_title = 4;
    string translated_summary = 5;
    string translated_body = 6;
    string translator_id = 7;
}

message GetTranslationRequest {
    string tenant_id = 1;
    string source_article_id = 2;
    string locale = 3;
}

message TranslationResponse {
    TranslationMessage translation = 1;
}

message TranslationMessage {
    string id = 1;
    string source_article_id = 2;
    string locale = 3;
    string translated_title = 4;
    string translated_summary = 5;
    string translated_body = 6;
    string translation_status = 7;
    string translator_id = 8;
    google.protobuf.Timestamp translated_at = 9;
}

message ListTranslationsRequest {
    string tenant_id = 1;
    string source_article_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListTranslationsResponse {
    repeated TranslationMessage translations = 1;
    string next_page_token = 2;
}

message UpdateTranslationRequest {
    string tenant_id = 1;
    string translation_id = 2;
    string translated_title = 3;
    string translated_summary = 4;
    string translated_body = 5;
    string translation_status = 6;
    int32 version = 7;
}

message AddRelatedArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
    string related_article_id = 3;
    string relationship_type = 4;
    int32 relevance_score = 5;
}

message RelatedArticleResponse {
    string id = 1;
    string related_article_id = 2;
    string relationship_type = 3;
}

message ListRelatedArticlesRequest {
    string tenant_id = 1;
    string article_id = 2;
}

message ListRelatedArticlesResponse {
    repeated RelatedArticleSummary related = 1;
}

message RelatedArticleSummary {
    string article_id = 1;
    string title = 2;
    string summary = 3;
    string relationship_type = 4;
    int32 relevance_score = 5;
}

message RemoveRelatedArticleRequest {
    string tenant_id = 1;
    string article_id = 2;
    string related_article_id = 3;
}

message CategoryMessage {
    string id = 1;
    string tenant_id = 2;
    string category_code = 3;
    string name = 4;
    string description = 5;
    string parent_category_id = 6;
    int32 sort_order = 7;
    int32 article_count = 8;
    string visibility = 9;
}

message CreateCategoryRequest {
    string tenant_id = 1;
    string category_code = 2;
    string name = 3;
    string description = 4;
    string parent_category_id = 5;
    int32 sort_order = 6;
    string visibility = 7;
}

message CategoryResponse {
    CategoryMessage category = 1;
}

message ListCategoriesRequest {
    string tenant_id = 1;
    string parent_category_id = 2;
    int32 page_size = 3;
    string page_token = 4;
}

message ListCategoriesResponse {
    repeated CategoryMessage categories = 1;
    string next_page_token = 2;
}

message GetCategoryRequest {
    string tenant_id = 1;
    string category_id = 2;
}

message UpdateCategoryRequest {
    string tenant_id = 1;
    string category_id = 2;
    string name = 3;
    string description = 4;
    string parent_category_id = 5;
    int32 sort_order = 6;
    string visibility = 7;
    int32 version = 8;
}

message WorkflowTaskMessage {
    string id = 1;
    string tenant_id = 2;
    string article_id = 3;
    int32 article_version = 4;
    string workflow_type = 5;
    string from_status = 6;
    string to_status = 7;
    string submitted_by = 8;
    string assigned_reviewer_id = 9;
    string review_comments = 10;
    string priority = 11;
    google.protobuf.Timestamp due_at = 12;
    string status = 13;
    google.protobuf.Timestamp completed_at = 14;
    google.protobuf.Timestamp created_at = 15;
}

message ListWorkflowTasksRequest {
    string tenant_id = 1;
    string assigned_reviewer_id = 2;
    string status = 3;
    int32 page_size = 4;
    string page_token = 5;
}

message ListWorkflowTasksResponse {
    repeated WorkflowTaskMessage tasks = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message GetWorkflowTaskRequest {
    string tenant_id = 1;
    string workflow_id = 2;
}

message WorkflowTaskResponse {
    WorkflowTaskMessage task = 1;
}

message CompleteWorkflowTaskRequest {
    string tenant_id = 1;
    string workflow_id = 2;
    string action = 3;
    string review_comments = 4;
}

message UsageAnalyticsResponse {
    int32 total_views = 1;
    int32 unique_views = 2;
    double avg_dwell_time_seconds = 3;
    int32 total_searches = 4;
    double search_success_rate = 5;
    repeated DailyUsage daily_usage = 6;
}

message DailyUsage {
    string date = 1;
    int32 views = 2;
    int32 searches = 3;
    int32 ratings = 4;
}

message GetUsageAnalyticsRequest {
    string tenant_id = 1;
    string article_id = 2;
    string date_from = 3;
    string date_to = 4;
}

message PopularArticlesResponse {
    repeated PopularArticle articles = 1;
}

message PopularArticle {
    string article_id = 1;
    string title = 2;
    int32 view_count = 3;
    double average_rating = 4;
}

message GetPopularArticlesRequest {
    string tenant_id = 1;
    string category_id = 2;
    string period = 3;
    int32 max_results = 4;
}

message RecordArticleViewRequest {
    string tenant_id = 1;
    string article_id = 2;
    string user_id = 3;
    string user_type = 4;
    string session_id = 5;
    string search_query = 6;
    int32 search_position = 7;
    int32 dwell_time_seconds = 8;
    string referrer_source = 9;
    string channel = 10;
}

message AddAttachmentRequest {
    string tenant_id = 1;
    string article_id = 2;
    string file_name = 3;
    string file_type = 4;
    int64 file_size_bytes = 5;
    bytes content = 6;
    string description = 7;
}

message AttachmentResponse {
    string id = 1;
    string article_id = 2;
    string file_name = 3;
    string file_type = 4;
    int64 file_size_bytes = 5;
}

message ListAttachmentsRequest {
    string tenant_id = 1;
    string article_id = 2;
}

message ListAttachmentsResponse {
    repeated AttachmentMessage attachments = 1;
}

message AttachmentMessage {
    string id = 1;
    string file_name = 2;
    string file_type = 3;
    int64 file_size_bytes = 4;
    string description = 5;
    google.protobuf.Timestamp created_at = 6;
}

message RemoveAttachmentRequest {
    string tenant_id = 1;
    string attachment_id = 2;
}

message AddTagsRequest {
    string tenant_id = 1;
    string article_id = 2;
    repeated string tags = 3;
    string tag_type = 4;
}

message TagsResponse {
    repeated string tags = 1;
}

message ListTagsRequest {
    string tenant_id = 1;
    string article_id = 2;
}

message TagsListResponse {
    repeated TagMessage tags = 1;
}

message TagMessage {
    string id = 1;
    string tag = 2;
    string tag_type = 3;
}

message RemoveTagRequest {
    string tenant_id = 1;
    string tag_id = 2;
}
```

---

## 6. Inter-Service Integration

### Inbound Dependencies

| Source Service | Integration Type | Purpose |
|---|---|---|
| Customer Service | REST/gRPC | Serve knowledge search results to agents and chatbot sessions |
| Digital Assistant | REST/gRPC | Retrieve articles for AI-powered answer generation and suggestions |
| HR Help Desk | REST/gRPC | Access HR-specific knowledge articles for employee self-service |
| Identity Service | REST | User authentication, authorization, and role-based access to articles |
| Notification Service | Async | Send review assignment notifications to knowledge reviewers |
| File Storage Service | REST | Store and retrieve article attachments and media files |
| Search Index Service | Async | Maintain full-text search index for article content |

### Outbound Events Consumed By

| Target Service | Event | Purpose |
|---|---|---|
| Customer Service | ArticlePublished | Update cached knowledge suggestions for agent console |
| Digital Assistant | ArticlePublished, ArticleUpdated | Refresh AI model knowledge base for accurate responses |
| HR Help Desk | ArticlePublished | Push new HR policy articles to employee help desk |
| Notification Service | ArticleSubmittedForReview, ReviewOverdue | Notify reviewers of pending and overdue review tasks |
| Search Index Service | ArticlePublished, ArticleUnpublished, ArticleUpdated | Trigger search index rebuild for changed articles |

---

## 7. Events

| Event | Payload | Description |
|---|---|---|
| `ArticleCreated` | `{ article_id, article_number, tenant_id, title, category_id, owner_id, status }` | Emitted when a new article draft is created |
| `ArticleSubmittedForReview` | `{ article_id, tenant_id, article_version, assigned_reviewer_id, review_deadline, priority }` | Emitted when an article is submitted for review |
| `ArticleApproved` | `{ article_id, tenant_id, article_version, reviewer_id, review_comments }` | Emitted when a reviewer approves an article |
| `ArticleRejected` | `{ article_id, tenant_id, article_version, reviewer_id, review_comments }` | Emitted when a reviewer rejects an article |
| `ArticlePublished` | `{ article_id, tenant_id, article_version, title, category_id, locale, effective_start_date }` | Emitted when an article is published and goes live |
| `ArticleUnpublished` | `{ article_id, tenant_id, reason, unpublished_by }` | Emitted when a published article is taken offline |
| `ArticleExpired` | `{ article_id, tenant_id, article_number, expired_at }` | Emitted when an article reaches its expiration date |
| `ArticleRated` | `{ article_id, tenant_id, user_id, user_type, rating, is_helpful, new_average_rating }` | Emitted when a user rates an article |
| `ArticleViewed` | `{ article_id, tenant_id, user_id, user_type, channel, search_query, dwell_time_seconds }` | Emitted when an article is viewed for analytics tracking |
| `TranslationCreated` | `{ translation_id, tenant_id, source_article_id, locale, translation_status }` | Emitted when a new article translation is created |
| `ReviewOverdue` | `{ article_id, tenant_id, workflow_id, assigned_reviewer_id, review_deadline }` | Emitted when a review task passes its deadline without completion |
| `KnowledgeGapIdentified` | `{ tenant_id, search_query, result_count, search_frequency }` | Emitted when searches repeatedly yield no useful results |
