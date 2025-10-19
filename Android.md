
```kotlin
implementation("androidx.datastore:datastore-preferences:1.1.7")
implementation("com.google.code.gson:gson:2.12.0")

import android.content.Context
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.material3.Button
import androidx.compose.material3.Checkbox
import androidx.compose.material3.Text
import androidx.compose.material3.TextField
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberCoroutineScope
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.style.TextDecoration
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.datastore.preferences.preferencesDataStore
import com.google.gson.Gson
import com.google.gson.reflect.TypeToken
import com.rakibulahasan.test_notes_app.ui.theme.Test_notes_appTheme
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.launch

val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "notes")

private val NOTES_KEY = stringPreferencesKey("notes_list")

// Save list
suspend fun saveNotes(context: Context, notes: List<Note>) {
    val json = Gson().toJson(notes)
    context.dataStore.edit { prefs ->
        prefs[NOTES_KEY] = json
    }
}

// Load list
fun loadNotes(context: Context): Flow<List<Note>> = context.dataStore.data
    .map { prefs ->
        prefs[NOTES_KEY]?.let { json ->
            val type = object : TypeToken<List<Note>>() {}.type
            Gson().fromJson<List<Note>>(json, type)
        } ?: emptyList()
    }

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            Test_notes_appTheme {
                NotesApp()
            }
        }
    }
}


@Composable
fun NotesApp() {
    val context = LocalContext.current
    var text by remember { mutableStateOf("") }
    var notes by remember { mutableStateOf(listOf<Note>()) }

    // Load saved notes on startup
    LaunchedEffect(Unit) {
        loadNotes(context).collect { savedNotes ->
            notes = savedNotes
        }
    }

    val scope = rememberCoroutineScope()

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        // ---- Input Field ----
        TextField(
            value = text,
            onValueChange = { text = it },
            modifier = Modifier.fillMaxWidth(),
            placeholder = { Text("Enter note...") }
        )

        Spacer(modifier = Modifier.height(8.dp))

        // ---- Add Button ----
        Button(
            onClick = {
                if (text.isNotBlank()) {
                    notes = notes + Note(text = text)
                    text = ""
                    scope.launch(Dispatchers.IO) {
                        saveNotes(context, notes)
                    }
                }
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("Add Note")
        }

         // Wirte ne
        Spacer(modifier = Modifier.height(16.dp))

        // ---- Notes List ----
        LazyColumn {
            items(notes.size) { index ->
                val note = notes[index]

                Row(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(vertical = 4.dp),
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    Checkbox(
                        checked = note.isDone,
                        onCheckedChange = { checked ->
                            notes = notes.toMutableList().also {
                                it[index] = it[index].copy(isDone = checked)
                            }
                            scope.launch(Dispatchers.IO) {
                                saveNotes(context, notes)
                            }
                        }
                    )

                    Text(
                        text = note.text,
                        style = if (note.isDone) {
                            TextStyle(textDecoration = TextDecoration.LineThrough)
                        } else {
                            TextStyle.Default
                        },
                        modifier = Modifier.weight(1f)
                    )
                }
            }
        }
    }
}

// ---- Model ----
data class Note(
    val text: String,
    val isDone: Boolean = false
)

@Preview
@Composable
fun NotesAppPreview(){
    Test_notes_appTheme {
        NotesApp()
    }
}
```
