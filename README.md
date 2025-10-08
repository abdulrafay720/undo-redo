# undo-redo
A simple, customizable, and completely free example of implementing Undo / Redo functionality in React Flow.
Unlike [React Flow’s](https://reactflow.dev/pro) official undo/redo feature — which is part of their paid Pro version — this repository provides a fully open-source, custom implementation you can use in any project.

This is not a plug-and-play library, but rather a set of clear, well-documented functions and hooks showing exactly how to build undo and redo functionality yourself.
You can copy the code, modify it to your needs, and integrate it directly into your React Flow setup — no package installation or subscription required.

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

- The implementation keeps a **Undo stack** and  **Redo stack**. 
- Each time nodes or edges change, a snapshot is saved in **Undo Stack**.
- It include add and delete node, add and delete edge, drag a node and save node data configuration.
- Undo and redo simply move backward or forward in that stack.


## 🧠 Understanding the Code

This section explains how the custom Undo/Redo logic in useWorkflowState works — step by step.
Set the undo and redo stacks

## 1️⃣ Set Up Undo and Redo Stacks
```
const [undoStack, setUndoStack] = useState<{ nodes: NodeType[]; edges: Edge[] }[]>([]);
const [redoStack, setRedoStack] = useState<{ nodes: NodeType[]; edges: Edge[] }[]>([]);
```
💡 Why:

- We need to maintain a history of previous graph states `undoStack` and future states `redoStack`.
- `undoStack` holds snapshots that allow us to go backward in history.
- `redoStack` holds states that were undone and can be reapplied when the user redoes an action.
- Each snapshot contains both nodes and edges arrays — representing the full flow state at that point

## 2️⃣ Define Refs to Track Special States
```
const isUndoRedoRef = useRef(false);
const isDraggingRef = useRef(false);
const isLoadingRef = useRef(false);
const isInitializedRef = useRef(false);
```

💡 Why:
- Refs are used instead of state because we don’t want these flags to trigger re-renders.
- They help avoid unnecessary undo/redo tracking during:
- Undo/Redo actions themselves `(isUndoRedoRef)`
- Drag movements `(isDraggingRef)`
- Initial loading or reset states `(isLoadingRef)`
- Initial setup before first render `(isInitializedRef)`
- This prevents duplicate entries in our history stack.

## 3️⃣ Initialize the Undo Stack Once
```
useEffect(() => {
  if (isUndoRedoRef.current || isLoadingRef.current) return;
  if (!isInitializedRef.current && nodes.length > 0 && undoStack.length === 0) {
    setUndoStack([{ nodes: [...nodes], edges: [...edges] }]);
    isInitializedRef.current = true;
  }
}, [nodes, edges, undoStack.length]);
```
💡 Why:
- We start with a baseline snapshot of the current flow.
- This ensures our undo stack isn’t empty and always has an initial “known good” state to fall back to.

## 4️⃣ Track Node and Edge Changes
```
useEffect(() => {
  if (!isInitializedRef.current) return;
  if (isUndoRedoRef.current || isLoadingRef.current || isDraggingRef.current) return;

  const sanitizeNodes = (list: NodeType[]) =>
    list.map((n) => ({
      ...n,
      selected: undefined,
      dragging: undefined,
      positionAbsolute: undefined,
    }));

  const timer = setTimeout(() => {
    setUndoStack((prev) => {
      const lastState = prev[prev.length - 1];
      const currentNodes = sanitizeNodes(nodes);
      const lastNodes = lastState ? sanitizeNodes(lastState.nodes) : [];
      const nodesChanged = JSON.stringify(currentNodes) !== JSON.stringify(lastNodes);
      const edgesChanged = JSON.stringify(edges) !== JSON.stringify(lastState?.edges || []);

      if (!nodesChanged && !edgesChanged) return prev;

      return [...prev, { nodes: [...nodes], edges: [...edges] }];
    });
  }, 300);

  return () => clearTimeout(timer);
}, [nodes, edges]);
```

💡 Why:

- We watch for actual changes in nodes or edges.
- A short 300ms delay prevents excessive state pushes during rapid drag events.
- We sanitize nodes to remove volatile properties (like selected, dragging) so they don’t cause false positives.
- When a real change occurs, we push the new state into the undo stack.

## 5️⃣ Reset History
```
const resetHistory = useCallback(() => {
  isLoadingRef.current = true;
  isInitializedRef.current = false;
  isUndoRedoRef.current = true;

  setTimeout(() => {
    setUndoStack([{ nodes: [...getNodes()], edges: [...getEdges()] }]);
    setRedoStack([]);
    isInitializedRef.current = true;
    isLoadingRef.current = false;
    isUndoRedoRef.current = false;
  }, 100);
}, [getNodes, getEdges]);
```

💡 Why:
- Use this function when saving a workflow by API so it resets the both stacks.
- When reloading or clearing a workflow, we must reset both stacks to avoid old history interfering.
- This creates a clean initial snapshot representing the current flow.

## 6️⃣ Undo Action
```
const undo = useCallback(() => {
  if (undoStack.length <= 1) return;

  isUndoRedoRef.current = true;
  const previousState = undoStack[undoStack.length - 2];
  const currentState = undoStack[undoStack.length - 1];

  setUndoStack((prev) => prev.slice(0, -1));
  setRedoStack((prev) => [...prev, currentState]);

  setNodes(previousState.nodes);
  setEdges(previousState.edges);

  setTimeout(() => {
    isUndoRedoRef.current = false;
  }, 50);
}, [undoStack, setNodes, setEdges]);
```
💡 Why:

- Removes the current state from undoStack.
- Moves it into redoStack so we can redo it later.
- Restores the previous state as the active one.
- isUndoRedoRef prevents this action from being tracked as a new change.

## 7️⃣ Redo Action
```
const redo = useCallback(() => {
  if (redoStack.length === 0) return;

  isUndoRedoRef.current = true;
  const nextState = redoStack[redoStack.length - 1];

  setRedoStack((prev) => prev.slice(0, -1));
  setUndoStack((prev) => [...prev, nextState]);

  setNodes(nextState.nodes);
  setEdges(nextState.edges);

  setTimeout(() => {
    isUndoRedoRef.current = false;
  }, 50);
}, [redoStack, setNodes, setEdges]);
```
💡 Why:

- Pops a state from redoStack.
- Pushes it back to undoStack (so we can undo the redo).
- Reapplies that state visually in React Flow.
- Uses setTimeout to ensure React updates finish before resetting the flag.

## 8️⃣ Node and Edge Operations
```
const handleAddNodes = useCallback(
  (newNode: NodeType) => {
    addNodes(newNode);
    setNodes((nds) => [...nds, newNode]);
    setIsSaved(false);
  },
  [addNodes, setNodes]
);

const handleDeleteNode = useCallback(
  (nodeId: string) => {
    const children = nodes.filter((n) => n.parentId === nodeId).map((n) => n.id);
    const toDelete = [nodeId, ...children];
    setNodes((nds) => nds.filter((n) => !toDelete.includes(n.id)));
    setEdges((eds) =>
      eds.filter((e) => !toDelete.includes(e.source) && !toDelete.includes(e.target))
    );
    setIsSaved(false);
  },
  [nodes, setNodes, setEdges]
);
```
💡 Why:
- These methods manage node creation and deletion, marking the workflow as unsaved.
- Every call indirectly triggers the history watcher, creating a new undo checkpoint.

## 9️⃣Keyboard Shortcuts
```
    useEffect(() => {
        const handleKeyDown = (e: KeyboardEvent) => {
            const isUndo = (e.metaKey || e.ctrlKey) && e.key === "z";
            const isRedo =
                (e.metaKey || e.ctrlKey) && e.shiftKey && e.key === "Z";

            if (isUndo) {
                e.preventDefault();
                undo();
            } else if (isRedo) {
                e.preventDefault();
                redo();
            }
        };

        window.addEventListener("keydown", handleKeyDown);
        return () => window.removeEventListener("keydown", handleKeyDown);
    }, [undo, redo]);
```
`ctrl` + `z` for undo and `ctrl` + `Shift` + `z` for redo in Windows and `Win` + `z` for undo and `Win` + `Shift` + `z` for redo in Mac.

## 🔟 Buttons for Undo and Redo

```
<Button
      onClick={undo}
      variant="ghost"
      disabled={!canUndo}
      className={`border p-2 ${
      canUndo ? "border-red-400" : "border-input"
      }`}
>
    <Undo className="h-4" />
</Button>
<Button
      disabled={!canRedo}
      onClick={redo}
      variant="ghost"
      className={`border p-2 ${
      canRedo ? "border-red-400" : "border-input"
      }`}
>
      <Redo className="h-4" />
</Button>
```
In last use handleAddNode and handleDeleteNode functions in ReactFlow Component

## 🧩 Final Notes

This implementation is a **fully custom setup**, carefully designed with every important detail in mind — from React Flow’s internal state handling and update timing to preventing duplicate history entries and managing drag operations.  
It’s built to work **smoothly and reliably** with complex workflows while staying **easy to customize** for your own project needs.

You can easily:
- 🧠 Extend it to track selections, viewport, or custom node data  
- 🎨 Adjust the history depth or debounce timing  
- ⌨️ Add keyboard shortcuts like <kbd>Ctrl</kbd> + <kbd>Z</kbd> / <kbd>Ctrl</kbd> + <kbd>Y</kbd>  
- 🧩 Integrate it into any React Flow-based visual editor or builder  

React Flow has since introduced **its own Undo/Redo features** in newer versions — but this repository remains a **fully open, transparent, and dependency-free** example showing exactly how to build such functionality yourself.  

This project is **free to use, modify, and learn from** — whether you’re building an automation tool, workflow builder, or node-based editor.

---

### 💬 Support & Collaboration

If you need help implementing or customizing it for your project, feel free to reach out or open an issue — I’ll be happy to help!

👤 **Author:** [@Abdul Rafay](https://github.com/abdulrafay720)  
📫 **Contact:** Open a GitHub issue or Mail me directly for collaboration or support at rafay12105@gmail.com.

---

## ⭐ Give It a Star

If this project helped you, consider giving it a **star** on GitHub — it helps others discover it and supports continued improvements!  
👉 [Star this repo](https://github.com/abdulrafay720/undo-redo/stargazers)

## 🪪 License

This project is licensed under the **MIT License** — see the [LICENSE](./LICENSE) file for details.

---

Built with ❤️ by [Abdul Rafay](https://github.com/abdulrafay720)
