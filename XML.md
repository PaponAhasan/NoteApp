```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp"
    tools:context=".MainActivity">

    <EditText
        android:id="@+id/inputNote"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Enter note..." />

    <Button
        android:id="@+id/btnAdd"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Add Note" />

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:paddingTop="8dp" />

</LinearLayout>
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal"
    android:padding="8dp">

    <CheckBox
        android:id="@+id/checkBox"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

    <TextView
        android:id="@+id/textNote"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:textSize="16sp"
        android:paddingStart="8dp" />
</LinearLayout>
```

```kotlin
data class Note(
    val text: String,
    val isDone: Boolean = false
)

class NoteAdapter(
    private var notes: MutableList<Note>,
    private val onChecked: (Int, Boolean) -> Unit
) : RecyclerView.Adapter<NoteAdapter.NoteViewHolder>() {

    inner class NoteViewHolder(val view: View) : RecyclerView.ViewHolder(view) {
        val checkBox: CheckBox = view.findViewById(R.id.checkBox)
        val textView: TextView = view.findViewById(R.id.textNote)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): NoteViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_note, parent, false)
        return NoteViewHolder(view)
    }

    override fun onBindViewHolder(holder: NoteViewHolder, position: Int) {
        val note = notes[position]
        holder.checkBox.isChecked = note.isDone
        holder.textView.text = note.text

        holder.textView.paint.isStrikeThruText = note.isDone

        holder.checkBox.setOnCheckedChangeListener { _, isChecked ->
            onChecked(position, isChecked)
        }
    }

    override fun getItemCount() = notes.size

    fun updateList(newList: List<Note>) {
        notes.clear()
        notes.addAll(newList)
        notifyDataSetChanged()
    }
}
```

```kotlin
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "notes")
private val NOTES_KEY = stringPreferencesKey("notes")

suspend fun saveNotes(context: Context, notes: List<Note>) {
    val serialized = notes.joinToString("|") { "${it.text}::${it.isDone}" }
    context.dataStore.edit { prefs ->
        prefs[NOTES_KEY] = serialized
    }
}

fun loadNotes(context: Context): Flow<List<Note>> {
    return context.dataStore.data.map { prefs ->
        prefs[NOTES_KEY]
            ?.split("|")
            ?.filter { it.isNotBlank() }
            ?.map {
                val parts = it.split("::")
                Note(parts[0], parts.getOrNull(1)?.toBoolean() ?: false)
            } ?: emptyList()
    }
}
```

```kotlin
class MainActivity : AppCompatActivity() {

    private lateinit var adapter: NoteAdapter
    private val notes = mutableListOf<Note>()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val input = findViewById<EditText>(R.id.inputNote)
        val btnAdd = findViewById<Button>(R.id.btnAdd)
        val recyclerView = findViewById<RecyclerView>(R.id.recyclerView)

        adapter = NoteAdapter(notes) { position, checked ->
            notes[position] = notes[position].copy(isDone = checked)
            adapter.updateList(notes)
            saveNotesToStorage()
        }

        recyclerView.layoutManager = LinearLayoutManager(this)
        recyclerView.adapter = adapter

        btnAdd.setOnClickListener {
            val text = input.text.toString()
            if (text.isNotBlank()) {
                notes.add(Note(text))
                adapter.updateList(notes)
                saveNotesToStorage()
                input.text.clear()
            }
        }

        lifecycleScope.launch {
            loadNotes(this@MainActivity).collect { loadedNotes ->
                notes.clear()
                notes.addAll(loadedNotes)
                adapter.updateList(notes)
            }
        }
    }

    private fun saveNotesToStorage() {
        lifecycleScope.launch {
            saveNotes(this@MainActivity, notes)
        }
    }
}
```


