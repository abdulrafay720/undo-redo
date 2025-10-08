# undo-redo
A simple, customizable example of implementing Undo / Redo functionality in React Flow . Includes the core logic and helper functions you can copy or adapt for your own projects — no package installation required.


This repository contains a **custom implementation** of Undo and Redo functionality for [React Flow](https://reactflow.dev/).  
It’s not a library — it’s a working example and custom functions you can copy into your own project to add state history and undo/redo logic.

## 🎯 Purpose

React Flow's undo/redo functions are paid, so this project demonstrates how to:

- Track flow state changes (nodes, edges, positions)
- Save state history efficiently
- Undo and redo changes using keyboard shortcuts and buttons
- Keep your app’s state in sync with React Flow

---

## ⚙️ What’s Inside

- A **custom `undo` and `redo` functiions**
- Example usage integrated with `ReactFlow`
- Helper functions for handling node and edge updates, adds and deletes
- Comments explaining how it works internally

---

## 🧩 How It Works

The implementation keeps a **Undo stack** and  **Redo stack**. 
Each time nodes or edges change, a snapshot is saved in **Undo Stack**.
It include add and delete node, add and delete edge, drag a node and save node data configuration.
Undo and redo simply move backward or forward in that stack.


🧠 Understanding the Code

This section explains how the custom Undo/Redo logic in useWorkflowState works — step by step.
Set the undo and redo stacks
```
const [undoStack, setUndoStack] = useState<{ nodes: NodeType[]; edges: Edge[] }[]>([]);
const [redoStack, setRedoStack] = useState<{ nodes: NodeType[]; edges: Edge[] }[]>([]);
```


