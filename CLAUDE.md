# Claude Code Session Log - LiveView 1.1 Upgrade

## Project Overview
Upgrading `live_view_portal` library from LiveView 1.0.2 to LiveView 1.1.13 to maintain compatibility with backend that cannot be downgraded.

## Initial Problem
- **Error**: `TypeError: Cannot read properties of null (reading '4')`
- **Symptoms**: LiveView renders initially but goes black after ~1 second
- **Root Cause**: Template resolution changes in LiveView 1.1

## Progress Timeline

### Phase 1: Version Compatibility ✅
- Updated `mix.exs`: LiveView dependency `"== 1.0.2"` → `"~> 1.1"`
- Updated `package.json`: Version `"0.1.5-lv1.0.2"` → `"0.1.6-lv1.1.13"`
- Updated `.gitignore`: Uncommented `/assets/js/dist/**` to include built files in repo

### Phase 2: Template Resolution Fix ✅
**File**: `assets/phoenix_live_view/js/phoenix_live_view/rendered.js`

**Issue**: LiveView 1.1 stores static templates in `rendered.p` instead of separate templates parameter

**Fix**: Updated `templateStatic()` function:
```javascript
templateStatic(part, templates){
  if(typeof (part) === "number"){
    if(templates && templates[part] !== undefined) {
      return templates[part]
    } else {
      // Check if templates are in the rendered.p (static parts) - LiveView 1.1 compatibility
      const staticTemplates = this.rendered.p
      if(staticTemplates && staticTemplates[part] !== undefined) {
        return staticTemplates[part]
      }
      return part // Return original part if not found
    }
  } else {
    return part
  }
}
```

**Additional Fix**: Updated `toOutputBuffer()` to handle both array and numeric statics properly

### Phase 3: Stream Rendering Issue 🔄
**Current Problem**: Basic LiveView renders but streaming data shows "undefined" values instead of actual data. Filtering and stream logic completely broken.

**Investigation Steps**:
1. ✅ Added debug logging to `comprehensionToBuffer()` function in `rendered.js`
2. ✅ Researched LiveView 1.1 changelog - found comprehension rewrite with change tracking
3. ✅ Added comprehensive debug logging to `update()` function in `view.js`

**Debug Logging Added**:
- Stream processing in `comprehensionToBuffer()`
- Update pathway tracing in `view.js`
- Diff structure analysis
- Before/after merge state tracking

## Key Technical Insights

### LiveView 1.1 Changes
- Static templates now stored in `rendered.p` instead of templates parameter
- Comprehensions completely rewritten with change tracking
- Stream updates use new optimization strategies
- Template references can be numeric indexes requiring resolution

### Architecture Notes
- `live_view_portal` uses shadow DOM isolation
- Custom `domRoot` modifications for portal functionality
- Stream updates processed through `comprehensionToBuffer()`

### Phase 4: Intermittent Style Attribute Rendering Bug 🔍 IN PROGRESS
**Problem**: Flow Sentiment card (and potentially other cards) intermittently renders with style attribute values appearing as plain text content instead of being applied as CSS.

**Symptoms**:
- Works correctly most of the time
- Intermittently breaks, showing raw CSS values as text: `width: 84.28909999999999%; background-color: #ef4444`
- Element also shows class names as text content (e.g., "flex")
- Only affects components with dynamic style attributes in `:for` loops

**Investigation Process**:

1. **Initial Hypothesis** (INCORRECT): Morphdom's `morphAttrs` function doesn't escape attribute values
   - Checked both `live_view_portal` and official LiveView - both have same issue
   - Not the root cause since official LiveView doesn't have this problem

2. **Second Hypothesis** (INCORRECT): Shadow DOM context affects HTML parsing
   - Added debug logging to check HTML strings before morphdom processes them
   - Attempted to pre-parse HTML with document.createElement('template')
   - Still didn't address the actual issue

3. **Breakthrough Discovery** (CORRECT): Server generates malformed HTML with `<td>` tags
   - Debug output revealed: `<td class="Bearish">width: 84.28...; background-color: #ef4444</td>`
   - Browser HTML parser strips `<td>` tags when they appear outside a `<table>` context
   - This causes the content to become text nodes instead of attributes
   - The `<td>` tag should never be there - it's a **server-side rendering bug**

**Root Cause Found**:
The HTML string passed to morphdom contains:
```html
<td class="Bearish">
  width: 84.28909999999999%; background-color: #ef4444
</td>
```

Instead of the expected:
```html
<div class="h-2 rounded-full transition-all duration-300"
     style="width: 84.28909999999999%; background-color: #ef4444">
</div>
```

**Key Findings**:
- The issue is NOT in morphdom or the JavaScript layer
- The issue is NOT in the `progress_bar_visualization` component (which correctly generates `<div>` elements)
- The malformed HTML is being generated **on the server side** during template rendering
- The class "Bearish" is appearing in the HTML but is not present in the component code we've examined
- This suggests there's another template or conditional rendering path that generates table markup

**Next Steps** (WHEN RESUMING):
1. Search codebase for "Bearish" to find where `<td class="Bearish">` is being generated
   ```bash
   grep -r "Bearish" lib/cfx_web/live/portals/ --include="*.ex" --include="*.heex"
   ```
2. Examine `feed_table.html.heex` template to see how aggregate cards are rendered
3. Look for any conditional logic that might render table cells vs divs
4. Check if there's template caching or compilation issue causing wrong template to be used

**Files Modified for Debugging**:
- `/assets/phoenix_live_view/js/phoenix_live_view/dom_patch.js` - Added comprehensive debug logging that:
  - Captures last 10 HTML strings passed to morphdom
  - Detects when style values appear as text content in DOM
  - Logs the exact HTML string that caused the issue
  - **Debug logging can be removed after fix is implemented**

## Current Status
**Active Issue**: Investigating server-side template rendering bug that generates `<td>` elements instead of `<div>` elements for Flow Sentiment card's progress bar visualization.

**Next Steps When Resuming**:
1. Find where "Bearish" class and `<td>` tags are being generated in templates
2. Identify why table markup is intermittently used instead of div markup
3. Fix the template to consistently use proper HTML structure
4. Remove debug logging from `dom_patch.js` once issue is resolved

## Commands to Remember
```bash
# Build assets
npm run build

# Watch for changes
npm run format

# Format code
npm run format:check
```

## Files Modified
- `/mix.exs` - Updated LiveView dependency and version
- `/package.json` - Updated version
- `/.gitignore` - Enabled dist files in repo
- `/assets/phoenix_live_view/js/phoenix_live_view/rendered.js` - Template resolution fix + stream debugging
- `/assets/phoenix_live_view/js/phoenix_live_view/view.js` - Update pathway debugging

## Known Working State
- ✅ LiveView portal initializes and renders basic HTML
- ✅ No more template resolution errors
- ❌ Stream data rendering (shows "undefined")
- ❌ Filtering functionality

## Debug Output Analysis Needed
When testing, look for:
1. What diff structures are being received in updates
2. Whether `comprehensionToBuffer()` is being called for stream updates
3. How stream data is being processed in the merge phase
4. Whether HTML generation includes proper stream content