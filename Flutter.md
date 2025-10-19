```dart
  provider: ^6.1.5+1
  shared_preferences: ^2.2.2 # Persistence

class Note {
  final String text;
  final bool isDone;

  const Note({
    required this.text,
    this.isDone = false,
  });

  // Method to create a new Note instance with updated values
  Note copyWith({
    String? text,
    bool? isDone,
  }) {
    return Note(
      text: text ?? this.text,
      isDone: isDone ?? this.isDone,
    );
  }

  // Convert a Note object into a Map (JSON)
  Map<String, dynamic> toJson() {
    return {
      'text': text,
      'isDone': isDone,
    };
  }

  // Create a Note object from a Map (JSON)
  factory Note.fromJson(Map<String, dynamic> json) {
    return Note(
      text: json['text'] as String,
      isDone: json['isDone'] as bool? ?? false,
    );
  }
}


import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';

import 'Note.dart';

class NoteData extends ChangeNotifier {
  static const _notesKey = 'notes_list';

  List<Note> _notes = [];
  List<Note> get notes => _notes;

  NoteData() {
    _loadNotes();
  }

  // Load notes from SharedPreferences
  Future<void> _loadNotes() async {
    final prefs = await SharedPreferences.getInstance();
    final jsonStringList = prefs.getStringList(_notesKey) ?? [];

    _notes = jsonStringList.map((jsonString) {
      final jsonMap = jsonDecode(jsonString);
      return Note.fromJson(jsonMap as Map<String, dynamic>);
    }).toList();

    // Notify listeners after loading data from storage
    notifyListeners();
  }

  // Save notes to SharedPreferences
  Future<void> _saveNotes() async {
    final prefs = await SharedPreferences.getInstance();
    // Convert List<Note> to List<JSON String>
    final jsonStringList = _notes.map((note) => jsonEncode(note.toJson())).toList();

    await prefs.setStringList(_notesKey, jsonStringList);
  }

  // --- State Manipulation Methods ---
  // Add a new note
  void addNote(String text) {
    if (text.trim().isEmpty) return;

    final newNote = Note(text: text.trim());
    _notes.add(newNote);

    // 1. Notify listeners to rebuild UI
    notifyListeners();
    // 2. Persist the change
    _saveNotes();
  }

  // Toggle the 'isDone' status of a note
  void toggleNote(int index) {
    if (index < 0 || index >= _notes.length) return;

    final currentNote = _notes[index];
    // Use copyWith to create an immutable update
    _notes[index] = currentNote.copyWith(isDone: !currentNote.isDone);

    // 1. Notify listeners
    notifyListeners();
    // 2. Persist the change
    _saveNotes();
  }
}



import 'package:flutter/material.dart';

import 'package:provider/provider.dart';
import 'note_data.dart';


class NotesApp extends StatelessWidget {
  const NotesApp({super.key});

  @override
  Widget build(BuildContext context) {
    // Wrap the entire app with ChangeNotifierProvider to make NoteData available
    return ChangeNotifierProvider(
      create: (context) => NoteData(),
      child: MaterialApp(
        title: 'Flutter Notes App',
        theme: ThemeData(
          useMaterial3: true,
          colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal).copyWith(background: Colors.white),
        ),
        home: const NotesScreen(),
      ),
    );
  }
}

class NotesScreen extends StatefulWidget {
  const NotesScreen({super.key});

  @override
  State<NotesScreen> createState() => _NotesScreenState();
}

class _NotesScreenState extends State<NotesScreen> {
  final TextEditingController _controller = TextEditingController();

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    // Use Consumer to listen for changes from the NoteData provider
    return Consumer<NoteData>(
      builder: (context, noteData, child) {
        return Scaffold(
          appBar: AppBar(
            title: const Text('To-Do Notes App'),
            backgroundColor: Theme.of(context).colorScheme.primary,
            foregroundColor: Colors.white,
          ),
          body: Padding(
            padding: const EdgeInsets.all(16.0),
            child: Column(
              children: <Widget>[
                // ---- Input Field ----
                TextField(
                  controller: _controller,
                  decoration: const InputDecoration(
                    labelText: 'Enter note...',
                    border: OutlineInputBorder(),
                  ),
                ),
                const SizedBox(height: 8.0),

                // ---- Add Button ----
                SizedBox(
                  width: double.infinity,
                  child: ElevatedButton(
                    onPressed: () {
                      // Call the Provider method to add a note
                      noteData.addNote(_controller.text);
                      _controller.clear();
                    },
                    child: const Text('Add Note'),
                  ),
                ),
                const SizedBox(height: 16.0),

                // ---- Notes List ----
                Expanded(
                  child: ListView.builder(
                    itemCount: noteData.notes.length,
                    itemBuilder: (context, index) {
                      final note = noteData.notes[index];
                      return ListTile(
                        leading: Checkbox(
                          value: note.isDone,
                          onChanged: (bool? checked) {
                            // Call the Provider method to toggle status
                            noteData.toggleNote(index);
                          },
                        ),
                        title: Text(
                          note.text,
                          style: TextStyle(
                            fontSize: 16,
                            // Apply line-through decoration conditionally
                            decoration: note.isDone
                                ? TextDecoration.lineThrough
                                : TextDecoration.none,
                            color: note.isDone ? Colors.grey : Colors.black,
                          ),
                        ),
                        onTap: () {
                          // Allow tapping the ListTile to also toggle
                          noteData.toggleNote(index);
                        },
                      );
                    },
                  ),
                ),
              ],
            ),
          ),
        );
      },
    );
  }
}

```
