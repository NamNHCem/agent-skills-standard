---
name: Android XML Resources
description: Resource management, theming, and localization for Android XML.
metadata:
  labels: [android, xml, resources, themes, strings, colors]
  triggers:
    files: ['**/res/values/*.xml', '**/res/drawable/*.xml']
    keywords: [string, color, style, theme, drawable]
---

# Android XML Resources

## **Priority: P1 (OPERATIONAL)**

Proper resource organization, theming, and localization for Android applications.

## String Resources

### strings.xml
```xml
<resources>
    <!-- Simple strings -->
    <string name="app_name">My App</string>
    <string name="welcome_message">Welcome to My App!</string>
    
    <!-- Formatted strings -->
    <string name="welcome_user">Welcome, %1$s!</string>
    <string name="items_count">You have %d items</string>
    
    <!-- Plurals -->
    <plurals name="items">
        <item quantity="one">%d item</item>
        <item quantity="other">%d items</item>
    </plurals>
    
    <!-- String arrays -->
    <string-array name="categories">
        <item>Technology</item>
        <item>Science</item>
        <item>Arts</item>
    </string-array>
</resources>
```

### Usage in Kotlin
```kotlin
// Simple string
val appName = getString(R.string.app_name)

// Formatted string
val welcome = getString(R.string.welcome_user, userName)
val count = getString(R.string.items_count, itemCount)

// Plurals
val itemsText = resources.getQuantityString(R.plurals.items, count, count)

// String array
val categories = resources.getStringArray(R.array.categories)
```

## Color Resources

### colors.xml
```xml
<resources>
    <!-- Material Design Colors -->
    <color name="purple_200">#FFBB86FC</color>
    <color name="purple_500">#FF6200EE</color>
    <color name="purple_700">#FF3700B3</color>
    
    <!-- Semantic Colors -->
    <color name="color_primary">@color/purple_500</color>
    <color name="color_on_primary">#FFFFFFFF</color>
    <color name="color_background">#FFFFFFFF</color>
    <color name="color_error">#FFB00020</color>
    
    <!-- With Alpha -->
    <color name="overlay_dark">#80000000</color> <!-- 50% black -->
</resources>
```

## Themes & Styles

### themes.xml
```xml
<resources>
    <!-- Base Theme -->
    <style name="Theme.MyApp" parent="Theme.Material3.DayNight.NoActionBar">
        <item name="colorPrimary">@color/purple_500</item>
        <item name="colorPrimaryVariant">@color/purple_700</item>
        <item name="colorOnPrimary">@color/white</item>
        <item name="colorSecondary">@color/teal_200</item>
        <item name="android:statusBarColor">@color/purple_700</item>
    </style>
    
    <!-- Custom Styles -->
    <style name="Widget.MyApp.Button" parent="Widget.Material3.Button">
        <item name="android:textAllCaps">false</item>
        <item name="cornerRadius">8dp</item>
    </style>
    
    <style name="TextAppearance.MyApp.Title" parent="TextAppearance.Material3.TitleLarge">
        <item name="android:textSize">24sp</item>
        <item name="android:textColor">?attr/colorOnBackground</item>
    </style>
</resources>
```

### Dark Theme (themes.xml in values-night)
```xml
<resources>
    <style name="Theme.MyApp" parent="Theme.Material3.DayNight.NoActionBar">
        <item name="colorPrimary">@color/purple_200</item>
        <item name="colorPrimaryVariant">@color/purple_700</item>
        <item name="colorOnPrimary">@color/black</item>
        <item name="android:windowBackground">@color/dark_background</item>
    </style>
</resources>
```

## Drawable Resources

### Vector Drawable
```xml
<!-- drawable/ic_home.xml -->
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24"
    android:tint="?attr/colorControlNormal">
    <path
        android:fillColor="@android:color/white"
        android:pathData="M10,20v-6h4v6h5v-8h3L12,3 2,12h3v8z"/>
</vector>
```

### Shape Drawable
```xml
<!-- drawable/bg_rounded.xml -->
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <solid android:color="@color/purple_500" />
    <corners android:radius="16dp" />
    <padding
        android:left="16dp"
        android:top="8dp"
        android:right="16dp"
        android:bottom="8dp" />
</shape>
```

### Selector Drawable
```xml
<!-- drawable/button_selector.xml -->
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_pressed="true">
        <shape android:shape="rectangle">
            <solid android:color="@color/purple_700" />
            <corners android:radius="8dp" />
        </shape>
    </item>
    <item>
        <shape android:shape="rectangle">
            <solid android:color="@color/purple_500" />
            <corners android:radius="8dp" />
        </shape>
    </item>
</selector>
```

## Dimension Resources

### dimens.xml
```xml
<resources>
    <!-- Spacing -->
    <dimen name="spacing_xs">4dp</dimen>
    <dimen name="spacing_small">8dp</dimen>
    <dimen name="spacing_medium">16dp</dimen>
    <dimen name="spacing_large">24dp</dimen>
    <dimen name="spacing_xl">32dp</dimen>
    
    <!-- Text Sizes -->
    <dimen name="text_size_caption">12sp</dimen>
    <dimen name="text_size_body">14sp</dimen>
    <dimen name="text_size_title">20sp</dimen>
    <dimen name="text_size_headline">24sp</dimen>
    
    <!-- Component Sizes -->
    <dimen name="button_height">48dp</dimen>
    <dimen name="icon_size">24dp</dimen>
</resources>
```

## Localization

### Default (values/strings.xml)
```xml
<resources>
    <string name="welcome">Welcome</string>
    <string name="settings">Settings</string>
</resources>
```

### Vietnamese (values-vi/strings.xml)
```xml
<resources>
    <string name="welcome">Chào mừng</string>
    <string name="settings">Cài đặt</string>
</resources>
```

### French (values-fr/strings.xml)
```xml
<resources>
    <string name="welcome">Bienvenue</string>
    <string name="settings">Paramètres</string>
</resources>
```

## Configuration Qualifiers

### Screen Density
- `drawable-mdpi/` - Medium density (160dpi)
- `drawable-hdpi/` - High density (240dpi)
- `drawable-xhdpi/` - Extra-high density (320dpi)
- `drawable-xxhdpi/` - Extra-extra-high density (480dpi)
- `drawable-xxxhdpi/` - Extra-extra-extra-high density (640dpi)

### Screen Size
- `layout-small/` - Small screens
- `layout-normal/` - Normal screens
- `layout-large/` - Large screens (tablets)
- `layout-xlarge/` - Extra-large screens

### Orientation
- `layout-land/` - Landscape orientation
- `layout-port/` - Portrait orientation

## Anti-Patterns

- **No Hardcoded Strings**: Always use string resources
- **No Hardcoded Colors**: Use color resources
- **No Hardcoded Dimensions**: Use dimen resources
- **No Missing Translations**: Provide fallback strings
- **No PNG for Icons**: Use vector drawables

## Reference & Examples

See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

android-xml/layouts | android-xml/best-practices
