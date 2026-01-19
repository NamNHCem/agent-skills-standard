---
name: Android XML Layouts
description: Best practices for XML layout design and ViewBinding.
metadata:
  labels: [android, xml, layouts, viewbinding, constraintlayout]
  triggers:
    files: ['**/res/layout/*.xml']
    keywords: [ConstraintLayout, LinearLayout, FrameLayout, ViewBinding]
---

# Android XML Layouts

## **Priority: P1 (OPERATIONAL)**

Modern XML layout patterns with ConstraintLayout and ViewBinding.

## Layout Types

### ConstraintLayout (Preferred)
```xml
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    
    <TextView
        android:id="@+id/titleText"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:text="@string/title"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />
    
    <Button
        android:id="@+id/actionButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/action"
        app:layout_constraintTop_toBottomOf="@id/titleText"
        app:layout_constraintEnd_toEndOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

### Guidelines & Barriers
```xml
<androidx.constraintlayout.widget.ConstraintLayout>
    
    <!-- Guideline at 50% -->
    <androidx.constraintlayout.widget.Guideline
        android:id="@+id/guideline"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        app:layout_constraintGuide_percent="0.5" />
    
    <!-- Barrier -->
    <androidx.constraintlayout.widget.Barrier
        android:id="@+id/barrier"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:barrierDirection="end"
        app:constraint_referenced_ids="text1,text2" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

## ViewBinding

### Enable ViewBinding
```kotlin
// build.gradle.kts
android {
    buildFeatures {
        viewBinding = true
    }
}
```

### Activity Usage
```kotlin
class MainActivity : AppCompatActivity() {
    
    private lateinit var binding: ActivityMainBinding
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
        
        binding.actionButton.setOnClickListener {
            binding.titleText.text = "Clicked!"
        }
    }
}
```

### Fragment Usage
```kotlin
class HomeFragment : Fragment() {
    
    private var _binding: FragmentHomeBinding? = null
    private val binding get() = _binding!!
    
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentHomeBinding.inflate(inflater, container, false)
        return binding.root
    }
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.recyclerView.adapter = myAdapter
    }
    
    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null // Prevent memory leaks
    }
}
```

### RecyclerView ViewHolder
```kotlin
class UserViewHolder(
    private val binding: ItemUserBinding
) : RecyclerView.ViewHolder(binding.root) {
    
    fun bind(user: User) {
        binding.nameText.text = user.name
        binding.emailText.text = user.email
        binding.root.setOnClickListener {
            // Handle click
        }
    }
    
    companion object {
        fun from(parent: ViewGroup): UserViewHolder {
            val binding = ItemUserBinding.inflate(
                LayoutInflater.from(parent.context),
                parent,
                false
            )
            return UserViewHolder(binding)
        }
    }
}
```

## Layout Best Practices

### Avoid Deep Nesting
```xml
<!-- ❌ Bad - Deep nesting -->
<LinearLayout>
    <LinearLayout>
        <LinearLayout>
            <TextView />
        </LinearLayout>
    </LinearLayout>
</LinearLayout>

<!-- ✅ Good - Flat hierarchy -->
<ConstraintLayout>
    <TextView
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent" />
</ConstraintLayout>
```

### Use merge Tag
```xml
<!-- layout/custom_toolbar.xml -->
<merge xmlns:android="http://schemas.android.com/apk/res/android">
    <ImageView
        android:id="@+id/icon"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />
    
    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />
</merge>
```

### Include Layouts
```xml
<androidx.constraintlayout.widget.ConstraintLayout>
    
    <include
        android:id="@+id/toolbar"
        layout="@layout/toolbar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_constraintTop_toTopOf="parent" />
    
</androidx.constraintlayout.widget.ConstraintLayout>
```

## Dimensions & Spacing

### Use dp for Sizes
```xml
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:padding="16dp"
    android:textSize="14sp" /> <!-- sp for text -->
```

### Dimension Resources
```xml
<!-- values/dimens.xml -->
<resources>
    <dimen name="spacing_small">8dp</dimen>
    <dimen name="spacing_medium">16dp</dimen>
    <dimen name="spacing_large">24dp</dimen>
    <dimen name="text_size_body">14sp</dimen>
    <dimen name="text_size_title">20sp</dimen>
</resources>

<!-- Usage -->
<TextView
    android:padding="@dimen/spacing_medium"
    android:textSize="@dimen/text_size_body" />
```

## Anti-Patterns

- **No findViewById**: Use ViewBinding
- **No Deep Nesting**: Use ConstraintLayout
- **No Hardcoded Strings**: Use string resources
- **No Hardcoded Colors**: Use color resources
- **No match_parent with ConstraintLayout**: Use 0dp

## Reference & Examples

See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

android-xml/resources | kotlin/best-practices
