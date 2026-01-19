---
name: Android XML Best Practices
description: Best practices for Android XML development including RecyclerView, Fragments, and Lifecycle.
metadata:
  labels: [android, xml, recyclerview, fragments, lifecycle]
  triggers:
    files: ['**/*.kt', '**/*.xml']
    keywords: [RecyclerView, Fragment, Activity, Lifecycle, ViewModel]
---

# Android XML Best Practices

## **Priority: P1 (OPERATIONAL)**

Modern patterns for Android XML-based development.

## RecyclerView

### Adapter Pattern
```kotlin
class UserAdapter(
    private val onItemClick: (User) -> Unit
) : RecyclerView.Adapter<UserAdapter.UserViewHolder>() {
    
    private var users = listOf<User>()
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder {
        return UserViewHolder.from(parent)
    }
    
    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        holder.bind(users[position], onItemClick)
    }
    
    override fun getItemCount() = users.size
    
    fun submitList(newUsers: List<User>) {
        users = newUsers
        notifyDataSetChanged() // Use DiffUtil instead
    }
    
    class UserViewHolder private constructor(
        private val binding: ItemUserBinding
    ) : RecyclerView.ViewHolder(binding.root) {
        
        fun bind(user: User, onItemClick: (User) -> Unit) {
            binding.nameText.text = user.name
            binding.emailText.text = user.email
            binding.root.setOnClickListener { onItemClick(user) }
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
}
```

### ListAdapter with DiffUtil
```kotlin
class UserListAdapter(
    private val onItemClick: (User) -> Unit
) : ListAdapter<User, UserListAdapter.UserViewHolder>(UserDiffCallback()) {
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder {
        return UserViewHolder.from(parent)
    }
    
    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        holder.bind(getItem(position), onItemClick)
    }
    
    class UserViewHolder(
        private val binding: ItemUserBinding
    ) : RecyclerView.ViewHolder(binding.root) {
        
        fun bind(user: User, onItemClick: (User) -> Unit) {
            binding.user = user
            binding.root.setOnClickListener { onItemClick(user) }
            binding.executePendingBindings()
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
}

class UserDiffCallback : DiffUtil.ItemCallback<User>() {
    override fun areItemsTheSame(oldItem: User, newItem: User): Boolean {
        return oldItem.id == newItem.id
    }
    
    override fun areContentsTheSame(oldItem: User, newItem: User): Boolean {
        return oldItem == newItem
    }
}
```

## Fragment Best Practices

### Fragment Lifecycle
```kotlin
class HomeFragment : Fragment() {
    
    private var _binding: FragmentHomeBinding? = null
    private val binding get() = _binding!!
    
    private val viewModel: HomeViewModel by viewModels()
    
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
        setupUI()
        observeViewModel()
    }
    
    private fun setupUI() {
        binding.recyclerView.apply {
            adapter = userAdapter
            layoutManager = LinearLayoutManager(context)
        }
    }
    
    private fun observeViewModel() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                when (state) {
                    is UiState.Loading -> showLoading()
                    is UiState.Success -> showData(state.data)
                    is UiState.Error -> showError(state.message)
                }
            }
        }
    }
    
    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null // Prevent memory leaks
    }
}
```

### Fragment Communication
```kotlin
// Using ViewModel
class SharedViewModel : ViewModel() {
    private val _selectedItem = MutableLiveData<Item>()
    val selectedItem: LiveData<Item> = _selectedItem
    
    fun selectItem(item: Item) {
        _selectedItem.value = item
    }
}

// Fragment A
class ListFragment : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()
    
    private fun onItemClick(item: Item) {
        sharedViewModel.selectItem(item)
    }
}

// Fragment B
class DetailFragment : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        sharedViewModel.selectedItem.observe(viewLifecycleOwner) { item ->
            displayItem(item)
        }
    }
}
```

## Activity Best Practices

### Activity with ViewModel
```kotlin
class MainActivity : AppCompatActivity() {
    
    private lateinit var binding: ActivityMainBinding
    private val viewModel: MainViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
        
        setupUI()
        observeViewModel()
    }
    
    private fun setupUI() {
        binding.toolbar.apply {
            setSupportActionBar(this)
            title = getString(R.string.app_name)
        }
        
        binding.fab.setOnClickListener {
            viewModel.onFabClick()
        }
    }
    
    private fun observeViewModel() {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    updateUI(state)
                }
            }
        }
    }
}
```

## Lifecycle-Aware Components

### Lifecycle Observer
```kotlin
class LocationObserver(
    private val lifecycle: Lifecycle,
    private val onLocationUpdate: (Location) -> Unit
) : DefaultLifecycleObserver {
    
    init {
        lifecycle.addObserver(this)
    }
    
    override fun onStart(owner: LifecycleOwner) {
        // Start location updates
    }
    
    override fun onStop(owner: LifecycleOwner) {
        // Stop location updates
    }
}

// Usage in Fragment
class MapFragment : Fragment() {
    private lateinit var locationObserver: LocationObserver
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        locationObserver = LocationObserver(viewLifecycleOwner.lifecycle) { location ->
            updateMap(location)
        }
    }
}
```

## Navigation Component

### Navigation Graph
```xml
<!-- res/navigation/nav_graph.xml -->
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">
    
    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment"
        android:label="Home">
        <action
            android:id="@+id/action_home_to_details"
            app:destination="@id/detailsFragment" />
    </fragment>
    
    <fragment
        android:id="@+id/detailsFragment"
        android:name="com.example.DetailsFragment"
        android:label="Details">
        <argument
            android:name="userId"
            app:argType="string" />
    </fragment>
</navigation>
```

### Safe Args Navigation
```kotlin
// Navigate with arguments
val action = HomeFragmentDirections.actionHomeToDetails(userId = "123")
findNavController().navigate(action)

// Receive arguments
class DetailsFragment : Fragment() {
    private val args: DetailsFragmentArgs by navArgs()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        val userId = args.userId
    }
}
```

## Anti-Patterns

- **No Memory Leaks**: Always null binding in onDestroyView
- **No Context Leaks**: Use ApplicationContext for long-lived objects
- **No notifyDataSetChanged**: Use DiffUtil
- **No Direct Fragment Transactions**: Use Navigation Component
- **No Ignoring Lifecycle**: Use lifecycle-aware components

## Reference & Examples

See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

android-xml/layouts | android-xml/resources | kotlin/coroutines
