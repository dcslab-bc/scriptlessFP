# scriptlessFP
CSS Fingerprinting tracker

# CSS Fingerprinting Defense PoC: Tainted Flag Patch

## Overview

This Proof of Concept (PoC) implements a "tainted flag" mechanism to detect when JavaScript accesses computed styles that were influenced by privacy-sensitive CSS media queries. This is a first step toward defending against CSS-based fingerprinting attacks.

## Privacy-Sensitive Media Queries

The following media queries can reveal user preferences and are tracked:

| Media Query | Information Leaked |
|------------|-------------------|
| `prefers-color-scheme` | Light/dark mode preference |
| `prefers-reduced-motion` | Motion sensitivity (accessibility) |
| `prefers-contrast` | Contrast preference (accessibility) |
| `forced-colors` | High contrast mode (accessibility) |
| `prefers-reduced-data` | Data saver mode |
| `prefers-reduced-transparency` | Transparency preference (accessibility) |
| `inverted-colors` | Color inversion (accessibility) |

## Attack Vector

CSS fingerprinting works by:
1. Creating CSS rules with privacy-sensitive media queries
2. Using `getComputedStyle()` to detect which rules were applied
3. Building a fingerprint from the user's preferences

Example attack:
```html
<style>
  .probe { width: 100px; }
  @media (prefers-color-scheme: dark) {
    .probe { width: 200px; }
  }
</style>
<div class="probe" id="target"></div>
<script>
  // Fingerprinting: detect if user prefers dark mode
  const width = getComputedStyle(document.getElementById('target')).width;
  const prefersDark = width === '200px';
</script>
```

## Implementation Architecture

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                          CSS Style Resolution Flow                            │
├───────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐    │
│  │ MediaQueryEvaluator│   │   RuleFeatureSet   │   │    StyleEngine     │    │
│  │                    │   │                    │   │                    │    │
│  │ EvalFeature()      │──▶│ MediaQueryResult   │──▶│ HasPrivacySensitive│    │
│  │ detects privacy    │   │ Flags.is_privacy   │   │ MediaQueries()     │    │
│  │ sensitive query    │   │ _sensitive = true  │   │                    │    │
│  └────────────────────┘   └────────────────────┘   └─────────┬──────────┘    │
│                                                              │               │
│                                                              ▼               │
│  ┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐    │
│  │    MatchResult     │◀──│ElementRuleCollector│◀──│   StyleResolver    │    │
│  │                    │   │                    │   │                    │    │
│  │ SetAffectedBy      │   │ MutableMatched     │   │ MatchAllRules()    │    │
│  │ PrivacySensitive   │   │ Result()           │   │ checks flag &      │    │
│  │ MediaQuery()       │   │                    │   │ propagates         │    │
│  └─────────┬──────────┘   └────────────────────┘   └────────────────────┘    │
│            │                                                                  │
│            ▼                                                                  │
│  ┌────────────────────┐   ┌────────────────────────────┐                     │
│  │   ComputedStyle    │   │ CSSComputedStyleDeclaration│                     │
│  │                    │   │                            │                     │
│  │ AffectedByPrivacy  │──▶│ GetPropertyCSSValue()      │                     │
│  │ SensitiveMedia     │   │ checks tainted flag        │                     │
│  │ Query()            │   │ and logs warning           │                     │
│  └────────────────────┘   └────────────────────────────┘                     │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

## Modified Files

### 1. `core/style/computed_style_extra_fields.json5`
Added monotonic flag to track privacy-sensitive media query influence:
```json
{
  name: "AffectedByPrivacySensitiveMediaQuery",
  field_template: "monotonic_flag",
  default_value: "false",
  reset_on_new_style: true,
  custom_compare: true,
}
```

### 2. `core/css/resolver/media_query_result.h`
Added `is_privacy_sensitive` field to `MediaQueryResultFlags`:
```cpp
struct MediaQueryResultFlags {
  bool is_viewport_dependent = false;
  bool is_device_dependent = false;
  bool is_privacy_sensitive = false;  // NEW
  // ...
};
```

### 3. `core/css/media_query_evaluator.cc`
Modified `EvalFeature()` to detect privacy-sensitive media features:
```cpp
// Check for privacy-sensitive media features
static const char* const kPrivacySensitiveFeatures[] = {
    "prefers-color-scheme",
    "prefers-reduced-motion",
    "prefers-contrast",
    "forced-colors",
    "prefers-reduced-data",
    "prefers-reduced-transparency",
    "inverted-colors",
};
```

### 4. `core/css/resolver/match_result.h`
Added tracking methods:
```cpp
void SetAffectedByPrivacySensitiveMediaQuery() {
  affected_by_privacy_sensitive_media_query_ = true;
}
bool AffectedByPrivacySensitiveMediaQuery() const {
  return affected_by_privacy_sensitive_media_query_;
}
```

### 5. `core/css/rule_feature_set.h`
Added helper methods to access privacy-sensitive media query flags at document level:
```cpp
const MediaQueryResultFlags& GetMediaQueryResultFlags() const {
  return media_query_result_flags_;
}
// [PoC] CSS Fingerprinting Defense: Check if any privacy-sensitive media
// queries were evaluated.
bool HasPrivacySensitiveMediaQueries() const {
  return media_query_result_flags_.is_privacy_sensitive;
}
```

### 6. `core/css/element_rule_collector.h`
Added mutable accessor to allow setting flags on MatchResult:
```cpp
MatchResult& MutableMatchedResult() { return result_; }
```

### 7. `core/css/style_engine.h`
Added public method to expose privacy-sensitive media query check (since `GetRuleFeatureSet()` is private):
```cpp
// [PoC] CSS Fingerprinting Defense: Check if any privacy-sensitive media
// queries were evaluated in the document's stylesheets.
bool HasPrivacySensitiveMediaQueries() {
  DCHECK(global_rule_set_);
  UpdateActiveStyle();
  return global_rule_set_->GetRuleFeatureSet()
      .HasPrivacySensitiveMediaQueries();
}
```

### 8. `core/css/resolver/style_resolver.cc`
Two modifications:

**a) In `MatchAllRules()`** - Check document-level privacy-sensitive media query flag and propagate to MatchResult:
```cpp
// [PoC] CSS Fingerprinting Defense: Check if the document has any
// privacy-sensitive media queries and propagate to MatchResult.
if (GetDocument().GetStyleEngine().HasPrivacySensitiveMediaQueries()) {
  collector.MutableMatchedResult().SetAffectedByPrivacySensitiveMediaQuery();
}
```

**b) In `ApplyBaseStyleNoCache()`** - Propagate flag from MatchResult to ComputedStyle:
```cpp
// PoC: Propagate privacy-sensitive media query tainted flag
if (match_result.AffectedByPrivacySensitiveMediaQuery()) {
  state.StyleBuilder().SetAffectedByPrivacySensitiveMediaQuery();
}
```

### 9. `core/css/css_computed_style_declaration.cc`
Check flag in `GetPropertyCSSValue()` and log warning:
```cpp
if (style->AffectedByPrivacySensitiveMediaQuery()) {
  LOG(WARNING) << "[CSS Fingerprinting PoC] getComputedStyle() called on "
               << "element whose style is affected by privacy-sensitive "
               << "media query. Property: " << property_name
               << ", Element tag: " << styled_element->tagName();
}
```

## Building and Testing

### Build
```bash
autoninja -C out/Default chrome
```

### Test
Create a test HTML file:
```html
<!DOCTYPE html>
<html>
<head>
  <style>
    .test-element {
      background-color: white;
      color: black;
    }
    @media (prefers-color-scheme: dark) {
      .test-element {
        background-color: black;
        color: white;
      }
    }
  </style>
</head>
<body>
  <div class="test-element" id="target">Test Element</div>
  <script>
    // This should trigger the PoC warning
    const style = getComputedStyle(document.getElementById('target'));
    console.log('Background:', style.backgroundColor);
    console.log('Color:', style.color);
  </script>
</body>
</html>
```

Run Chrome with logging:
```bash
out/Default/chrome --enable-logging=stderr --v=1 /path/to/test.html 2>&1 | grep "CSS Fingerprinting"
```

Expected output:
```
[CSS Fingerprinting PoC] getComputedStyle() called on element whose style is affected by privacy-sensitive media query. Property: background-color, Element tag: DIV
[CSS Fingerprinting PoC] getComputedStyle() called on element whose style is affected by privacy-sensitive media query. Property: color, Element tag: DIV
```

## Future Work

1. **Noise Injection**: Instead of just logging, inject random values for properties affected by privacy-sensitive media queries
2. **Permission Prompt**: Ask user permission before allowing fingerprinting
3. **Metrics Collection**: Use `UseCounter` to measure prevalence of fingerprinting attempts in the wild
4. **Allowlist**: Allow trusted origins to access real values
5. **DevTools Integration**: Show warning in DevTools Console

## References

- [CSS-based Fingerprinting Research](https://example.com/css-fingerprinting)
- [Blink Style Architecture](https://chromium.googlesource.com/chromium/src/+/HEAD/third_party/blink/renderer/core/css/style-readme.md)
- [How Blink Works](https://docs.google.com/document/d/1aitSOucL0VHZa9Z2vbRJSyAIsAz24kX8LFByQ5xQnUg)
