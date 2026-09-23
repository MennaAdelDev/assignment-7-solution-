## Models

### Users Schema
```javascript
// FILE: models/user.model.js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  phone: { type: String, required: true },
  age: { type: Number, min: 18, max: 60 },
});

module.exports = mongoose.model("User", userSchema);
```

### Notes Schema (with custom "not entirely uppercase" title validator)
```javascript
// FILE: models/note.model.js
const mongoose = require("mongoose");

const noteSchema = new mongoose.Schema(
  {
    title: {
      type: String,
      required: true,
      validate: {
        validator: function (value) {
          return value !== value.toUpperCase();
        },
        message: (props) => `${props.value} must not be entirely uppercase`,
      },
    },
    content: { type: String, required: true },
    userId: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
  },
  { timestamps: true } // adds createdAt & updatedAt
);

module.exports = mongoose.model("Note", noteSchema);
```

---

## A- User APIs

### Q1 — Signup
```javascript
// FILE: controllers/user.controller.js
const bcrypt = require("bcrypt");
const User = require("../models/user.model");

async function signup(req, res) {
  try {
    const { name, email, password, phone, age } = req.body;
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: "Email already exists." });
    }
    const hashedPassword = await bcrypt.hash(password, 10);
    await User.create({ name, email, password: hashedPassword, phone, age });
    res.status(201).json({ message: "User added successfully." });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q2 — Login
```javascript
// FILE: controllers/user.controller.js
async function login(req, res) {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: "Invalid email or password" });
    }
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ message: "Invalid email or password" });
    }
    const { password: _, ...userData } = user.toObject();
    res.json(userData);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q3 — Update logged-in user
```javascript
// FILE: controllers/user.controller.js
async function updateUser(req, res) {
  try {
    const { id } = req.params;
    const { password, ...updateData } = req.body;

    const user = await User.findById(id);
    if (!user) {
      return res.status(404).json({ message: "User not found" });
    }

    if (updateData.email && updateData.email !== user.email) {
      const emailExists = await User.findOne({ email: updateData.email });
      if (emailExists) {
        return res.status(400).json({ message: "Email already exists." });
      }
    }

    Object.assign(user, updateData);
    await user.save();
    res.json({ message: "User updated" });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q4 — Delete logged-in user
```javascript
// FILE: controllers/user.controller.js
async function deleteUser(req, res) {
  try {
    const { id } = req.query;
    const user = await User.findByIdAndDelete(id);
    if (!user) {
      return res.status(404).json({ message: "User not found" });
    }
    res.json({ message: "User deleted" });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q5 — Get logged-in user by id
```javascript
// FILE: controllers/user.controller.js
async function getUser(req, res) {
  try {
    const { id } = req.query;
    const user = await User.findById(id);
    if (!user) {
      return res.status(404).json({ message: "User not found" });
    }
    res.json(user);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}

module.exports = { signup, login, updateUser, deleteUser, getUser };
```

### routes/user.routes.js
```javascript
// FILE: routes/user.routes.js
const express = require("express");
const router = express.Router();
const { signup, login, updateUser, deleteUser, getUser } = require("../controllers/user.controller");

router.post("/users/signup", signup);
router.post("/users/login", login);
router.patch("/users/:id", updateUser);
router.delete("/users", deleteUser);
router.get("/users", getUser);

module.exports = router;
```

---

## B- Note APIs

### Q1 — Create a single note
```javascript
// FILE: controllers/note.controller.js
const Note = require("../models/note.model");

async function createNote(req, res) {
  try {
    const { id } = req.query;
    const { title, content } = req.body;
    await Note.create({ title, content, userId: id });
    res.status(201).json({ message: "Note created" });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q2 — Update a single note by id
```javascript
// FILE: controllers/note.controller.js
async function updateNote(req, res) {
  try {
    const { id } = req.query;
    const { noteId } = req.params;

    const note = await Note.findById(noteId);
    if (!note) {
      return res.status(404).json({ message: "Note not found" });
    }
    if (note.userId.toString() !== id) {
      return res.status(403).json({ message: "You are not the owner" });
    }

    Object.assign(note, req.body);
    await note.save();
    res.json({ message: "Note updated", note });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q3 — Replace entire note
```javascript
// FILE: controllers/note.controller.js
async function replaceNote(req, res) {
  try {
    const { id } = req.query;
    const { noteId } = req.params;

    const note = await Note.findById(noteId);
    if (!note) {
      return res.status(404).json({ message: "Note not found" });
    }
    if (note.userId.toString() !== id) {
      return res.status(403).json({ message: "You are not the owner" });
    }

    const { title, content, userId } = req.body;
    note.title = title;
    note.content = content;
    note.userId = userId;
    await note.save();
    res.json({ message: "Note updated", note });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q4 — Update the title of all notes for the logged-in user
```javascript
// FILE: controllers/note.controller.js
async function updateAllNotesTitle(req, res) {
  try {
    const { id } = req.query;
    const { title } = req.body;

    const result = await Note.updateMany({ userId: id }, { $set: { title } });
    if (result.matchedCount === 0) {
      return res.status(404).json({ message: "No note found" });
    }
    res.json({ message: "All notes updated" });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q6 — Delete a single note by id
```javascript
// FILE: controllers/note.controller.js
async function deleteNote(req, res) {
  try {
    const { id } = req.query;
    const { noteId } = req.params;

    const note = await Note.findById(noteId);
    if (!note) {
      return res.status(404).json({ message: "Note not found" });
    }
    if (note.userId.toString() !== id) {
      return res.status(403).json({ message: "You are not the owner" });
    }

    await Note.findByIdAndDelete(noteId);
    res.json({ message: "deleted", note });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q7 — Paginated list sorted by createdAt descending
```javascript
// FILE: controllers/note.controller.js
async function paginateSortNotes(req, res) {
  try {
    const { id } = req.query;
    const page = Number(req.query.page) || 1;
    const limit = Number(req.query.limit) || 10;

    const notes = await Note.find({ userId: id })
      .sort({ createdAt: -1 })
      .skip((page - 1) * limit)
      .limit(limit);

    res.json(notes);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q8 — Get a note by its id
```javascript
// FILE: controllers/note.controller.js
async function getNoteById(req, res) {
  try {
    const { id } = req.query;
    const note = await Note.findById(req.params.id);
    if (!note) {
      return res.status(404).json({ message: "Note not found" });
    }
    if (note.userId.toString() !== id) {
      return res.status(403).json({ message: "You are not the owner" });
    }
    res.json(note);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q9 — Get a note by its content
```javascript
// FILE: controllers/note.controller.js
async function getNoteByContent(req, res) {
  try {
    const { id, content } = req.query;
    const note = await Note.findOne({ userId: id, content });
    if (!note) {
      return res.status(404).json({ message: "No note found" });
    }
    res.json(note);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q10 — Get all notes with user info (populate)
```javascript
// FILE: controllers/note.controller.js
async function getNotesWithUser(req, res) {
  try {
    const { id } = req.query;
    const notes = await Note.find({ userId: id })
      .select("title userId createdAt")
      .populate("userId", "email -_id");
    res.json(notes);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q11 — Aggregation: notes with user info, searchable by title
```javascript
// FILE: controllers/note.controller.js
const mongoose = require("mongoose");

async function aggregateNotes(req, res) {
  try {
    const { id, title } = req.query;

    const match = { userId: new mongoose.Types.ObjectId(id) };
    if (title) {
      match.title = title;
    }

    const notes = await Note.aggregate([
      { $match: match },
      {
        $lookup: {
          from: "users",
          localField: "userId",
          foreignField: "_id",
          as: "user",
        },
      },
      { $unwind: "$user" },
      {
        $project: {
          _id: 0,
          title: 1,
          userId: 1,
          createdAt: 1,
          "user.name": 1,
          "user.email": 1,
        },
      },
    ]);

    res.json(notes);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}
```

### Q12 — Delete all notes for the logged-in user
```javascript
// FILE: controllers/note.controller.js
async function deleteAllNotes(req, res) {
  try {
    const { id } = req.query;
    await Note.deleteMany({ userId: id });
    res.json({ message: "Deleted" });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
}

module.exports = {
  createNote,
  updateNote,
  replaceNote,
  updateAllNotesTitle,
  deleteNote,
  paginateSortNotes,
  getNoteById,
  getNoteByContent,
  getNotesWithUser,
  aggregateNotes,
  deleteAllNotes,
};
```

### routes/note.routes.js
```javascript
// FILE: routes/note.routes.js
const express = require("express");
const router = express.Router();
const {
  createNote,
  updateNote,
  replaceNote,
  updateAllNotesTitle,
  deleteNote,
  paginateSortNotes,
  getNoteById,
  getNoteByContent,
  getNotesWithUser,
  aggregateNotes,
  deleteAllNotes,
} = require("../controllers/note.controller");

router.post("/notes", createNote);
router.patch("/notes/all", updateAllNotesTitle);
router.patch("/notes/:noteId", updateNote);
router.put("/notes/replace/:noteId", replaceNote);
router.get("/notes/paginate-sort", paginateSortNotes);
router.get("/notes/note-by-content", getNoteByContent);
router.get("/notes/note-with-user", getNotesWithUser);
router.get("/notes/aggregate", aggregateNotes);
router.get("/notes/:id", getNoteById);
router.delete("/notes/:noteId", deleteNote);
router.delete("/notes", deleteAllNotes);

module.exports = router;
```

---

## Bonus — Longest Common Prefix
```javascript
// FILE: bonus.js
var longestCommonPrefix = function (strs) {
  if (strs.length === 0) return "";

  let prefix = strs[0];

  for (let i = 1; i < strs.length; i++) {
    while (strs[i].indexOf(prefix) !== 0) {
      prefix = prefix.substring(0, prefix.length - 1);
      if (prefix === "") return "";
    }
  }

  return prefix;
};
```
