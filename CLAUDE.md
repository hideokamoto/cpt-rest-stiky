# CLAUDE.md - AI Assistant Guide for CPT Sticky Posts

## Project Overview

**CPT Sticky Posts** is a WordPress plugin that adds "sticky post" functionality to Custom Post Types (CPT) and exposes this feature through the WordPress REST API. The plugin is written in Japanese (カスタム投稿タイプ = Custom Post Type) and targets Japanese WordPress users.

**Primary Purpose:**
- Enable "stick to top" functionality for custom post types (WordPress core only supports this for standard posts)
- Provide REST API access to sticky post status
- Allow filtering and sorting posts by sticky status via REST API

**Tech Stack:**
- Pure PHP (7.4+)
- WordPress 5.0+
- No external dependencies or build process
- Single-file architecture

## Repository Structure

```
cpt-rest-stiky/
├── .git/                    # Git repository data
├── .gitignore              # WordPress-specific gitignore
├── LICENSE                 # GPL v2 or later license
├── README.md               # User-facing documentation (Japanese)
├── CLAUDE.md               # This file - AI assistant guide
└── cpt-rest-stiky.php      # Main plugin file (entire codebase)
```

**Important:** This is a single-file plugin with no build process, no package.json, no JavaScript files, and no external dependencies.

## Codebase Architecture

### Single Class Pattern

The entire plugin is implemented as a single PHP class `CPT_Sticky_Posts` that hooks into WordPress at various points.

**File:** `cpt-rest-stiky.php:15-276`

**Class Structure:**
```php
class CPT_Sticky_Posts {
    private array $target_post_types = [];  // Stores eligible post types

    public function __construct()           // Registers all WordPress hooks
    public function set_target_post_types() // Determines which CPTs to target

    // REST API Registration
    public function register_sticky_meta()
    public function register_rest_query_params()

    // REST API Callbacks
    public function get_sticky_field()
    public function update_sticky_field()
    public function filter_rest_query()
    public function add_collection_params()

    // Admin UI
    public function add_sticky_meta_box()
    public function render_sticky_meta_box()
    public function save_sticky_meta()
    public function add_sticky_column()
    public function render_sticky_column()
}
```

## Key Components

### 1. Target Post Type Detection (Line 36-49)

The plugin automatically targets all custom post types that meet these criteria:
- `public = true`
- `show_in_rest = true`
- `_builtin = false` (excludes WordPress core post types)

**Filter Hook:** Users can override this via `cpt_sticky_posts_target_types` filter.

### 2. Data Storage

**Meta Key:** `_cpt_is_sticky`
**Value Type:** Boolean (stored as '1' or '' in WordPress post meta)
**Scope:** Per-post metadata

### 3. REST API Integration

#### REST Field Registration (Line 68-76)
- Adds `sticky` field to REST API responses
- Type: boolean
- Contexts: view, edit
- Auth: `edit_posts` capability required for updates

#### Query Parameters (Line 173-186)
1. **sticky** (boolean, optional)
   - `true`: Returns only sticky posts
   - `false`: Returns only non-sticky posts

2. **sticky_first** (boolean, default: false)
   - `true`: Sorts sticky posts first, then by original order

#### Query Filter Logic (Line 110-168)

**For `sticky=true`:**
```php
meta_query: _cpt_is_sticky = '1'
```

**For `sticky=false`:**
```php
meta_query: OR (
    _cpt_is_sticky NOT EXISTS,
    _cpt_is_sticky = '',
    _cpt_is_sticky = '0'
)
```

**For `sticky_first=true`:**
```php
orderby: [
    'sticky_clause' => 'DESC',  // Sticky posts first
    {existing_orderby} => {existing_order}
]
```

### 4. Admin UI Components

#### Meta Box (Line 192-223)
- Location: Sidebar
- Priority: High
- Contains: Single checkbox with nonce protection
- Hook: `add_meta_boxes`

#### Admin Column (Line 257-272)
- Shows dashicons-sticky icon for sticky posts
- Shows "—" for non-sticky posts
- Hook: `manage_posts_columns` and `manage_posts_custom_column`

## Development Workflow

### Local Development Setup

1. **WordPress Installation Required**
   ```bash
   # This plugin must be placed in a WordPress installation
   # Typical path: /wp-content/plugins/cpt-sticky-posts/
   ```

2. **No Build Process**
   - No npm install needed
   - No compilation step
   - Direct PHP execution

3. **Testing Environment**
   - Requires WordPress 5.0+
   - Requires PHP 7.4+
   - Requires at least one custom post type with `show_in_rest = true`

### Making Changes

1. **Edit `cpt-rest-stiky.php` directly**
2. **Test in WordPress admin** - Check meta box appears
3. **Test REST API** - Use curl or Postman
4. **Verify permissions** - Test with different user roles

### Testing the Plugin

#### Manual Testing Checklist

**Admin UI:**
```bash
1. Activate plugin in WordPress
2. Navigate to custom post type edit screen
3. Verify "先頭に固定" meta box appears in sidebar
4. Check/uncheck and save
5. Verify sticky column shows dashicon in post list
```

**REST API:**
```bash
# Get all posts with sticky field
curl http://yoursite.local/wp-json/wp/v2/{post_type}

# Get only sticky posts
curl http://yoursite.local/wp-json/wp/v2/{post_type}?sticky=true

# Get posts with sticky first
curl http://yoursite.local/wp-json/wp/v2/{post_type}?sticky_first=true

# Update sticky status (requires authentication)
curl -X POST http://yoursite.local/wp-json/wp/v2/{post_type}/123 \
  -H "X-WP-Nonce: {nonce}" \
  -H "Content-Type: application/json" \
  -d '{"sticky": true}'
```

## WordPress Integration Points

### Hooks Used by This Plugin

| Hook | Type | Purpose | Line |
|------|------|---------|------|
| `init` | action | Set target post types | 23 |
| `rest_api_init` | action | Register REST meta and params | 24-25 |
| `add_meta_boxes` | action | Add sticky meta box | 26 |
| `save_post` | action | Save sticky meta value | 27 |
| `manage_posts_columns` | filter | Add sticky column | 28 |
| `manage_posts_custom_column` | action | Render sticky column | 29 |
| `rest_{post_type}_query` | filter | Modify REST queries | 100 |
| `rest_{post_type}_collection_params` | filter | Add query param schemas | 103 |

### Hooks Provided by This Plugin

| Hook | Type | Purpose | Usage |
|------|------|---------|-------|
| `cpt_sticky_posts_target_types` | filter | Override target post types | Line 48 |

**Example Usage:**
```php
add_filter( 'cpt_sticky_posts_target_types', function( $post_types ) {
    return [ 'news', 'event' ];  // Only target these types
} );
```

## Coding Conventions

### WordPress Coding Standards

This plugin follows WordPress coding standards:

1. **Naming Conventions:**
   - Class: `CPT_Sticky_Posts` (uppercase with underscores)
   - Methods: `snake_case`
   - Hooks: `lowercase_with_underscores`
   - Meta key: `_cpt_is_sticky` (leading underscore = hidden)

2. **Security:**
   - Nonce verification: Line 230-233
   - Capability checks: Line 241-243, Line 62-64
   - ABSPATH check: Line 11-13
   - Autosave protection: Line 236-238
   - Output escaping: Line 217, 220, 270

3. **Type Declarations:**
   - PHP 7.4+ syntax with type hints
   - Return types declared on all methods
   - Property types declared (Line 20)

4. **Internationalization:**
   - Text domain: `cpt-sticky-posts`
   - All strings wrapped in `__()` or `esc_html_e()`

5. **Code Organization:**
   - Logical method grouping
   - Clear comments for each section
   - PHPDoc blocks for complex methods

## Common Tasks

### Adding a New Feature

**Example: Add a "Featured" field alongside sticky**

1. **Register new meta field** (add after line 66):
```php
register_post_meta( $post_type, '_cpt_is_featured', [
    'type' => 'boolean',
    'single' => true,
    'default' => false,
    'show_in_rest' => true,
] );
```

2. **Add to meta box** (modify line 208-223)
3. **Update save method** (modify line 228-252)

### Modifying REST API Behavior

**Example: Change default sorting when sticky_first is true**

Modify `filter_rest_query()` at line 157-165:
```php
$args['orderby'] = [
    'sticky_clause' => 'DESC',
    'date' => 'DESC',  // Force date ordering
];
```

### Changing Target Post Types

**Option 1: Modify default behavior** (Line 38-42)
```php
$post_types = get_post_types( [
    'public' => true,
    'show_in_rest' => true,
    // Remove '_builtin' => false to include standard posts
], 'names' );
```

**Option 2: Use filter** (in theme's functions.php):
```php
add_filter( 'cpt_sticky_posts_target_types', function() {
    return [ 'custom_type_1', 'custom_type_2' ];
} );
```

## Performance Considerations

### Meta Queries Impact

- `sticky_first=true` uses `meta_query` which can be slow on large datasets
- WordPress doesn't index post meta by default
- Consider adding database indexes for `_cpt_is_sticky` on high-traffic sites

### Optimization Recommendations

1. **For large sites:** Add database index
```sql
ALTER TABLE wp_postmeta
ADD INDEX idx_cpt_sticky (meta_key, meta_value);
```

2. **Caching:** WordPress object cache will help
3. **Limit use:** Only use `sticky_first` when necessary

## AI Assistant Guidelines

### When Working with This Codebase

1. **Read Before Modifying:**
   - ALWAYS read `cpt-rest-stiky.php` before making changes
   - This is a single-file plugin - all code is in one place

2. **WordPress Context:**
   - This code runs within WordPress environment
   - WordPress functions (get_post_meta, add_action, etc.) are available
   - User capabilities and nonces are critical for security

3. **Language:**
   - User-facing text is in Japanese
   - Code comments and variables are in English
   - Maintain this bilingual pattern

4. **Testing:**
   - Changes cannot be fully tested without WordPress installation
   - Provide manual testing instructions
   - Suggest REST API curl commands for verification

5. **Security First:**
   - NEVER remove nonce checks
   - NEVER remove capability checks
   - ALWAYS escape output (esc_html, esc_attr, esc_url)
   - ALWAYS validate and sanitize input

6. **No Build Process:**
   - Don't suggest npm, webpack, or build tools
   - Don't create package.json
   - This is pure PHP - no transpilation needed

7. **WordPress Best Practices:**
   - Use WordPress functions over PHP equivalents
   - Follow WordPress coding standards
   - Maintain hook naming conventions
   - Keep text domain consistent

### Common Pitfalls to Avoid

1. **Don't assume post types exist** - Always check if `$target_post_types` is populated
2. **Don't break REST API compatibility** - Maintain backward compatibility for API clients
3. **Don't hardcode post types** - Use the filter system
4. **Don't forget nonce verification** - All form submissions need nonce checks
5. **Don't use PHP 8.0+ features** - Minimum version is 7.4

### Debugging Tips

1. **Enable WordPress debug mode:**
```php
// wp-config.php
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
```

2. **Check REST API schema:**
```bash
curl http://yoursite.local/wp-json/wp/v2/{post_type}?_fields=sticky
```

3. **Verify post meta:**
```php
// In WordPress debug console
get_post_meta(POST_ID, '_cpt_is_sticky', true);
```

4. **Check registered post types:**
```php
$types = get_post_types(['show_in_rest' => true], 'objects');
print_r($types);
```

## Git Workflow

### Branch Strategy

- Main branch: `main` (or unspecified)
- Feature branches: Should use prefix `claude/` and include session ID
- Example: `claude/add-claude-documentation-Mrtla`

### Commit Guidelines

1. **Keep commits focused** - One logical change per commit
2. **Write clear messages** - Explain what and why, not how
3. **Include session URL** - Add Claude session URL to commit messages
4. **Test before committing** - Verify changes work in WordPress

### Push Guidelines

- Always use: `git push -u origin <branch-name>`
- Branch must start with `claude/` and end with session ID
- Retry up to 4 times with exponential backoff on network errors

## File Modification History

- **Initial Creation:** 2024 (based on git log)
- **Last Updated:** Current session (check git log for details)

## Quick Reference

### Key Files
- **Main Plugin:** `cpt-rest-stiky.php` (only PHP file)
- **Documentation:** `README.md` (Japanese), `CLAUDE.md` (English)
- **License:** `LICENSE` (GPL v2+)

### Key Class Methods
- **Init:** `__construct()` - Registers all hooks
- **Core Logic:** `filter_rest_query()` - Handles REST API filtering
- **Data Access:** `get_sticky_field()`, `update_sticky_field()`
- **Admin UI:** `render_sticky_meta_box()`, `save_sticky_meta()`

### REST API Endpoints
- **Get posts:** `GET /wp-json/wp/v2/{post_type}`
- **Query sticky:** `GET /wp-json/wp/v2/{post_type}?sticky=true`
- **Sort sticky first:** `GET /wp-json/wp/v2/{post_type}?sticky_first=true`
- **Update sticky:** `POST /wp-json/wp/v2/{post_type}/{id}` with `{"sticky": true}`

### WordPress Requirements
- Version: 5.0+
- PHP: 7.4+
- Requires: Custom post types with `show_in_rest = true`

---

**Last Updated:** 2026-02-04
**Plugin Version:** 1.0.0
**For Questions:** Refer to README.md or WordPress Codex
