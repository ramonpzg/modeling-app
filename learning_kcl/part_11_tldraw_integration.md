# Part 11: Integrating KCL with tldraw (The Infinite Canvas)

You've learned KCL. You've built a Jupyter kernel. Now you'll do something nobody else has done: combine KCL with tldraw's infinite canvas. This opens up a new way to think about CAD programming: code and sketches living side by side, mixed fidelity, exploration and precision on the same surface.

By the end of this part, you'll have KCL code running inside tldraw shapes, with real-time execution and output.

## Why This Matters

Traditional CAD tools force a choice: either sketch (low fidelity, exploratory) or model (high precision, constrained). But design doesn't work that way. You sketch ideas, then make some precise while keeping others rough. You annotate precision with rough notes. You explore variations side by side.

tldraw's infinite canvas + KCL's parametric code = the best of both worlds.

## The Technology Stack

**tldraw** is a React-based infinite canvas library. It's what powers tldraw.com, but it's also a library you can embed:

```typescript
import { Tldraw } from '@tldraw/tldraw'

function App() {
  return <Tldraw />
}
```

That's it. You get an infinite canvas with drawing tools, shapes, text, images, and importantly: **custom shapes**.

**Custom shapes** are where the magic happens. You can create shapes that are full React components:

```typescript
class CodeEditorShape extends BaseBoxShapeUtil {
  // This shape is a code editor
  // It can execute code
  // It can display output
  // It's a fully interactive React component
}
```

KCL already runs in the browser (via WASM). The modeling-app is React + TypeScript. tldraw is React + TypeScript. Everything aligns.

## Project Setup

We'll build this as an extension to the modeling-app. First, explore the structure:

```bash
cd modeling-app
ls src/
```

Key directories:
- `src/components/` - React components
- `src/lang/` - KCL language integration
- `src/lib/` - Utilities and engine connection
- `src/wasm-lib/` - WASM bindings

Install tldraw:

```bash
cd modeling-app
npm install @tldraw/tldraw
```

## Creating a Basic tldraw Integration

Create `src/components/TldrawKclCanvas.tsx`:

```typescript
import { Tldraw } from '@tldraw/tldraw'
import '@tldraw/tldraw/tldraw.css'

export function TldrawKclCanvas() {
  return (
    <div style={{ position: 'fixed', inset: 0 }}>
      <Tldraw />
    </div>
  )
}
```

Add it to your app. In `src/App.tsx`, add a route:

```typescript
import { TldrawKclCanvas } from './components/TldrawKclCanvas'

// In your router config
{
  path: '/canvas',
  element: <TldrawKclCanvas />
}
```

Run the app:

```bash
npm run dev
```

Navigate to `/canvas`. You have an infinite canvas. You can draw, add shapes, move things around. But there's no KCL yet.

## Custom Shape: KCL Code Block

tldraw's custom shapes are defined with a utility class and a component. Create `src/shapes/KclCodeShape.tsx`:

```typescript
import {
  BaseBoxShapeUtil,
  HTMLContainer,
  TLBaseShape,
  T
} from '@tldraw/tldraw'
import { useEffect, useRef, useState } from 'react'

// Define the shape's data structure
export type KclCodeShape = TLBaseShape<
  'kcl-code',
  {
    w: number
    h: number
    code: string
  }
>

// Define the shape utility
export class KclCodeShapeUtil extends BaseBoxShapeUtil<KclCodeShape> {
  static override type = 'kcl-code' as const

  // Default props when creating a new shape
  getDefaultProps(): KclCodeShape['props'] {
    return {
      w: 400,
      h: 300,
      code: '// Write KCL code here\nwidth = 100\nheight = 50',
    }
  }

  // Render the shape
  component(shape: KclCodeShape) {
    return <KclCodeEditor shape={shape} />
  }

  // Indicator (shown when selected)
  indicator(shape: KclCodeShape) {
    return <rect width={shape.props.w} height={shape.props.h} />
  }
}

// The actual editor component
function KclCodeEditor({ shape }: { shape: KclCodeShape }) {
  const [code, setCode] = useState(shape.props.code)
  const [output, setOutput] = useState('')
  const [error, setError] = useState('')

  const handleExecute = async () => {
    try {
      // We'll implement this next
      const result = await executeKcl(code)
      setOutput(result.output)
      setError('')
    } catch (e) {
      setError(e.message)
      setOutput('')
    }
  }

  return (
    <HTMLContainer>
      <div style={{
        width: shape.props.w,
        height: shape.props.h,
        display: 'flex',
        flexDirection: 'column',
        background: '#1e1e1e',
        borderRadius: '8px',
        overflow: 'hidden',
      }}>
        {/* Code editor */}
        <textarea
          value={code}
          onChange={(e) => setCode(e.target.value)}
          style={{
            flex: 1,
            fontFamily: 'monospace',
            fontSize: '14px',
            padding: '12px',
            background: '#1e1e1e',
            color: '#d4d4d4',
            border: 'none',
            resize: 'none',
            outline: 'none',
          }}
        />

        {/* Execute button */}
        <button
          onClick={handleExecute}
          style={{
            padding: '8px 16px',
            background: '#007acc',
            color: 'white',
            border: 'none',
            cursor: 'pointer',
            fontWeight: 'bold',
          }}
        >
          Execute KCL
        </button>

        {/* Output area */}
        {output && (
          <div style={{
            padding: '12px',
            background: '#252526',
            color: '#d4d4d4',
            fontFamily: 'monospace',
            fontSize: '12px',
            maxHeight: '100px',
            overflow: 'auto',
          }}>
            {output}
          </div>
        )}

        {/* Error area */}
        {error && (
          <div style={{
            padding: '12px',
            background: '#5a1d1d',
            color: '#f48771',
            fontFamily: 'monospace',
            fontSize: '12px',
            maxHeight: '100px',
            overflow: 'auto',
          }}>
            {error}
          </div>
        )}
      </div>
    </HTMLContainer>
  )
}

// Placeholder for KCL execution (we'll implement this next)
async function executeKcl(code: string): Promise<{ output: string }> {
  return { output: 'KCL execution not yet implemented' }
}
```

Register the shape in your tldraw instance. Update `src/components/TldrawKclCanvas.tsx`:

```typescript
import { Tldraw } from '@tldraw/tldraw'
import { KclCodeShapeUtil } from '../shapes/KclCodeShape'

const customShapeUtils = [KclCodeShapeUtil]

export function TldrawKclCanvas() {
  return (
    <div style={{ position: 'fixed', inset: 0 }}>
      <Tldraw shapeUtils={customShapeUtils} />
    </div>
  )
}
```

## Adding a Tool to Create KCL Shapes

Users need a way to create KCL code shapes. Create a tool:

```typescript
// src/tools/KclCodeTool.tsx
import { StateNode, TLEventHandlers } from '@tldraw/tldraw'
import { KclCodeShape } from '../shapes/KclCodeShape'

export class KclCodeTool extends StateNode {
  static override id = 'kcl-code'

  override onEnter = () => {
    this.editor.setCursor({ type: 'cross', rotation: 0 })
  }

  override onPointerDown: TLEventHandlers['onPointerDown'] = (info) => {
    const { currentPagePoint } = info

    // Create a new KCL code shape at the click position
    this.editor.createShape<KclCodeShape>({
      type: 'kcl-code',
      x: currentPagePoint.x,
      y: currentPagePoint.y,
      props: {
        w: 400,
        h: 300,
        code: '// Write KCL code here\nwidth = 100\nheight = 50',
      },
    })

    // Switch back to select tool
    this.editor.setCurrentTool('select')
  }
}
```

Add the tool to tldraw:

```typescript
// src/components/TldrawKclCanvas.tsx
import { Tldraw } from '@tldraw/tldraw'
import { KclCodeShapeUtil } from '../shapes/KclCodeShape'
import { KclCodeTool } from '../tools/KclCodeTool'

const customShapeUtils = [KclCodeShapeUtil]
const customTools = [KclCodeTool]

export function TldrawKclCanvas() {
  return (
    <div style={{ position: 'fixed', inset: 0 }}>
      <Tldraw
        shapeUtils={customShapeUtils}
        tools={customTools}
        onMount={(editor) => {
          // Add toolbar button
          editor.updateInstanceState({ isToolLocked: false })
        }}
      />
    </div>
  )
}
```

Now you can create KCL code shapes on the canvas. But they don't execute yet.

## Executing KCL Code

KCL execution happens via the WASM library. Look at how the existing modeling-app does it:

```bash
grep -r "executeKcl\|execute_kcl" src/
```

You'll find the execution happens through a WASM module. Let's implement our `executeKcl` function properly:

```typescript
// src/lib/kclExecution.ts
import init, { execute_kcl } from 'kcl-wasm-lib'

let wasmInitialized = false

async function ensureWasmInitialized() {
  if (!wasmInitialized) {
    await init()
    wasmInitialized = true
  }
}

export async function executeKcl(code: string): Promise<{
  output: string
  variables: Record<string, any>
  errors: string[]
}> {
  await ensureWasmInitialized()

  try {
    const result = execute_kcl(code)

    // Parse the result (format depends on your WASM bindings)
    return {
      output: formatOutput(result),
      variables: result.variables || {},
      errors: result.errors || [],
    }
  } catch (e) {
    return {
      output: '',
      variables: {},
      errors: [e.message],
    }
  }
}

function formatOutput(result: any): string {
  const lines: string[] = []

  // Show variables
  if (result.variables) {
    for (const [name, value] of Object.entries(result.variables)) {
      lines.push(`${name} = ${JSON.stringify(value)}`)
    }
  }

  // Show geometry info
  if (result.geometry) {
    lines.push(`\nGenerated ${result.geometry.faces} faces, ${result.geometry.edges} edges`)
  }

  return lines.join('\n')
}
```

Update the shape to use this:

```typescript
// src/shapes/KclCodeShape.tsx
import { executeKcl } from '../lib/kclExecution'

function KclCodeEditor({ shape }: { shape: KclCodeShape }) {
  // ... previous code

  const handleExecute = async () => {
    try {
      const result = await executeKcl(code)

      if (result.errors.length > 0) {
        setError(result.errors.join('\n'))
        setOutput('')
      } else {
        setOutput(result.output)
        setError('')
      }
    } catch (e) {
      setError(e.message)
      setOutput('')
    }
  }

  // ... rest of component
}
```

## Real-time Execution

Right now, you click "Execute KCL" to run code. Let's make it run automatically as you type (with debouncing):

```typescript
import { useEffect, useRef, useState } from 'react'
import { debounce } from 'lodash'

function KclCodeEditor({ shape }: { shape: KclCodeShape }) {
  const [code, setCode] = useState(shape.props.code)
  const [output, setOutput] = useState('')
  const [error, setError] = useState('')

  // Create a debounced execution function
  const executeDebounced = useRef(
    debounce(async (codeToExecute: string) => {
      try {
        const result = await executeKcl(codeToExecute)

        if (result.errors.length > 0) {
          setError(result.errors.join('\n'))
          setOutput('')
        } else {
          setOutput(result.output)
          setError('')
        }
      } catch (e) {
        setError(e.message)
        setOutput('')
      }
    }, 500) // 500ms delay
  ).current

  // Execute when code changes
  useEffect(() => {
    executeDebounced(code)
  }, [code, executeDebounced])

  // ... rest of component
}
```

Now the code executes automatically as you type, with a 500ms delay to avoid running on every keystroke.

## Persisting Shape State

tldraw shapes need to persist their state. Update the shape props:

```typescript
// src/shapes/KclCodeShape.tsx

function KclCodeEditor({ shape }: { shape: KclCodeShape }) {
  const editor = useEditor()
  const [code, setCode] = useState(shape.props.code)

  const handleCodeChange = (newCode: string) => {
    setCode(newCode)

    // Update the shape's props so it persists
    editor.updateShape<KclCodeShape>({
      id: shape.id,
      type: 'kcl-code',
      props: {
        ...shape.props,
        code: newCode,
      },
    })
  }

  return (
    <HTMLContainer>
      <div style={{ /* ... */ }}>
        <textarea
          value={code}
          onChange={(e) => handleCodeChange(e.target.value)}
          style={{ /* ... */ }}
        />
        {/* ... rest of component */}
      </div>
    </HTMLContainer>
  )
}
```

Now when you save the tldraw file, your KCL code persists.

## Adding Syntax Highlighting

Plain textarea is ugly. Let's add CodeMirror (which the modeling-app already uses):

```bash
npm install @codemirror/state @codemirror/view @codemirror/lang-javascript
```

Create a CodeMirror wrapper:

```typescript
// src/components/KclCodeMirror.tsx
import { useEffect, useRef } from 'react'
import { EditorState } from '@codemirror/state'
import { EditorView, basicSetup } from '@codemirror/view'
import { javascript } from '@codemirror/lang-javascript'
import { oneDark } from '@codemirror/theme-one-dark'

interface Props {
  value: string
  onChange: (value: string) => void
  height: number
}

export function KclCodeMirror({ value, onChange, height }: Props) {
  const containerRef = useRef<HTMLDivElement>(null)
  const viewRef = useRef<EditorView | null>(null)

  useEffect(() => {
    if (!containerRef.current) return

    const startState = EditorState.create({
      doc: value,
      extensions: [
        basicSetup,
        javascript(), // Use JavaScript mode (close enough to KCL)
        oneDark,
        EditorView.updateListener.of((update) => {
          if (update.docChanged) {
            onChange(update.state.doc.toString())
          }
        }),
      ],
    })

    const view = new EditorView({
      state: startState,
      parent: containerRef.current,
    })

    viewRef.current = view

    return () => {
      view.destroy()
    }
  }, [])

  // Update when value changes externally
  useEffect(() => {
    if (viewRef.current && viewRef.current.state.doc.toString() !== value) {
      viewRef.current.dispatch({
        changes: {
          from: 0,
          to: viewRef.current.state.doc.length,
          insert: value,
        },
      })
    }
  }, [value])

  return (
    <div
      ref={containerRef}
      style={{
        height: `${height}px`,
        overflow: 'auto',
      }}
    />
  )
}
```

Use it in the shape:

```typescript
// src/shapes/KclCodeShape.tsx
import { KclCodeMirror } from '../components/KclCodeMirror'

function KclCodeEditor({ shape }: { shape: KclCodeShape }) {
  // ... state

  return (
    <HTMLContainer>
      <div style={{ /* ... */ }}>
        <KclCodeMirror
          value={code}
          onChange={handleCodeChange}
          height={shape.props.h - 100} // Leave room for button and output
        />
        {/* ... button and output */}
      </div>
    </HTMLContainer>
  )
}
```

Now you have proper syntax highlighting, autocomplete, and all CodeMirror features.

## Testing the Integration

1. Run the app: `npm run dev`
2. Navigate to `/canvas`
3. Click somewhere on the canvas (or add a toolbar button for the KCL tool)
4. A KCL code block appears
5. Type code:

```kcl
width = 100
height = 50
area = width * height
```

6. See the output update in real-time:

```
width = 100
height = 50
area = 5000
```

## What You Just Built

You've created:
- A custom tldraw shape for KCL code
- Real-time code execution
- Syntax highlighting with CodeMirror
- Persistent state (code saves with the canvas)
- Error handling and display

This is the foundation. In the next part, we'll add 3D geometry visualization.

## What You Just Learned

1. tldraw custom shapes are React components
2. Shape utilities define behavior (resize, serialize, etc.)
3. Tools let users create shapes
4. HTMLContainer lets you render arbitrary React inside shapes
5. KCL WASM bindings enable browser-side execution
6. Debouncing prevents excessive re-execution
7. CodeMirror integrates cleanly with React

## Exercises

1. **Add multiple code blocks**: Create several KCL code shapes and verify they execute independently.

2. **Add variable sharing**: Make one code block's variables available to another (requires global execution context).

3. **Add export button**: Let users export their KCL code to a `.kcl` file.

4. **Improve error display**: Show line numbers and syntax highlighting in errors.

5. **Add examples menu**: Pre-populate code blocks with example KCL programs (box, cylinder, bracket, etc.).

## Next Up

In Part 12, we'll add 3D geometry visualization. You'll create a geometry viewer shape that displays KCL's 3D output using Three.js. Code shapes will link to geometry shapes, and changes will propagate in real-time. You'll see your CAD models appearing on the infinite canvas.

The canvas is about to get three-dimensional.
