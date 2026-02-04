# Testing Guide for sticky_first Fix

## Issue Fixed
The `sticky_first=true` parameter was excluding posts without the `_cpt_is_sticky` meta field, causing legacy posts to disappear from results.

## Fix Applied
Changed the meta_query from using `compare => 'EXISTS'` to using OR logic that includes:
1. Posts with `_cpt_is_sticky = '1'` (sticky posts)
2. Posts with NO `_cpt_is_sticky` meta (legacy/existing posts)
3. Posts with `_cpt_is_sticky = ''` (empty)
4. Posts with `_cpt_is_sticky = '0'` (explicitly not sticky)

## Testing Steps

### Setup Test Data
Create test posts with different meta states:

```bash
# In WordPress environment
wp post create --post_type=your_cpt --post_title="Post 1: Sticky" --meta_input='{"_cpt_is_sticky": "1"}'
wp post create --post_type=your_cpt --post_title="Post 2: Not Sticky" --meta_input='{"_cpt_is_sticky": "0"}'
wp post create --post_type=your_cpt --post_title="Post 3: No Meta" # Legacy post without meta
wp post create --post_type=your_cpt --post_title="Post 4: Empty Meta" --meta_input='{"_cpt_is_sticky": ""}'
```

### Test sticky_first=true
```bash
# Should return ALL 4 posts with sticky post first
curl "https://your-site.com/wp-json/wp/v2/your_cpt?sticky_first=true"
```

**Expected Result:**
- ✓ All 4 posts are returned
- ✓ "Post 1: Sticky" appears first
- ✓ "Post 3: No Meta" is included (NOT excluded)
- ✓ Other posts follow in date order

### Test sticky=false (should still work)
```bash
# Should return only non-sticky posts (including posts without meta)
curl "https://your-site.com/wp-json/wp/v2/your_cpt?sticky=false"
```

**Expected Result:**
- ✓ Returns Posts 2, 3, 4 (excludes Post 1)
- ✓ "Post 3: No Meta" is included

### Test sticky=true (should still work)
```bash
# Should return only sticky posts
curl "https://your-site.com/wp-json/wp/v2/your_cpt?sticky=true"
```

**Expected Result:**
- ✓ Returns only Post 1
- ✓ Posts without meta are excluded (correct behavior)

## Verification Checklist
- [ ] Legacy posts without `_cpt_is_sticky` meta are included when `sticky_first=true`
- [ ] Sticky posts appear first in results
- [ ] Non-sticky posts follow in date order
- [ ] `sticky=true` still filters correctly
- [ ] `sticky=false` still filters correctly
- [ ] Combining `sticky_first` with other params (search, pagination) works correctly

## Code Reference
See `cpt-rest-stiky.php:149-187` for the implementation.
