# Part 12: 3D Geometry Visualization in tldraw

You've got KCL code running in tldraw shapes. Now you'll render the actual 3D geometry. This is where it gets real: write code on the left, see the CAD model on the right, all on the same infinite canvas.

By the end of this part, you'll have interactive 3D viewers as tldraw shapes, linked to code blocks, updating in real-time.

## The Challenge

KCL produces 3D geometry. tldraw is 2D canvas. How do you show 3D in 2D?

Answer: Embed a 3D renderer (Three.js) inside a tldraw shape. The shape is 2D (it has x, y, width, height), but its content is a 3D viewport.

Think of it like this: each geometry viewer shape is a window into 3D space. You can have multiple windows on the same canvas, each showing different angles or different models.

## Three.js Basics (Crash Course)

Three.js renders 3D graphics in the browser. The basic structure:

```typescript
import * as THREE from 'three'

// 1. Create a scene (the 3D world)
const scene = new THREE.Scene()

// 2. Create a camera (the viewpoint)
const camera = new THREE.PerspectiveCamera(
  75,                           // Field of view
  width / height,               // Aspect ratio
  0.1,                          // Near clipping plane
  1000                          // Far clipping plane
)
camera.position.z = 5

// 3. Create a renderer (draws the scene)
const renderer = new THREE.WebGLRenderer()
renderer.setSize(width, height)
document.body.appendChild(renderer.domElement)

// 4. Add objects to the scene
const geometry = new THREE.BoxGeometry(1, 1, 1)
const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 })
const cube = new THREE.Mesh(geometry, material)
scene.add(cube)

// 5. Render loop
function animate() {
  requestAnimationFrame(animate)
  cube.rotation.x += 0.01
  cube.rotation.y += 0.01
  renderer.render(scene, camera)
}
animate()
```

This creates a spinning green cube. We'll use this pattern for KCL geometry.

## Install Dependencies

```bash
npm install three @react-three/fiber @react-three/drei
```

- `three`: The core Three.js library
- `@react-three/fiber`: React renderer for Three.js (declarative)
- `@react-three/drei`: Useful helpers (camera controls, lighting, etc.)

## Creating the Geometry Viewer Shape

Create `src/shapes/GeometryViewerShape.tsx`:

```typescript
import {
  BaseBoxShapeUtil,
  HTMLContainer,
  TLBaseShape,
} from '@tldraw/tldraw'
import { Canvas } from '@react-three/fiber'
import { OrbitControls, Grid, Environment } from '@react-three/drei'
import * as THREE from 'three'

// Shape definition
export type GeometryViewerShape = TLBaseShape<
  'geometry-viewer',
  {
    w: number
    h: number
    geometryId: string | null  // Links to a KCL code shape
    meshData: MeshData | null   // The actual 3D data
  }
>

export interface MeshData {
  vertices: Float32Array
  indices: Uint32Array
  normals: Float32Array
}

// Shape utility
export class GeometryViewerShapeUtil extends BaseBoxShapeUtil<GeometryViewerShape> {
  static override type = 'geometry-viewer' as const

  getDefaultProps(): GeometryViewerShape['props'] {
    return {
      w: 400,
      h: 400,
      geometryId: null,
      meshData: null,
    }
  }

  component(shape: GeometryViewerShape) {
    return <GeometryViewer shape={shape} />
  }

  indicator(shape: GeometryViewerShape) {
    return <rect width={shape.props.w} height={shape.props.h} />
  }
}

// The actual viewer component
function GeometryViewer({ shape }: { shape: GeometryViewerShape }) {
  return (
    <HTMLContainer>
      <div
        style={{
          width: shape.props.w,
          height: shape.props.h,
          background: '#1a1a1a',
          borderRadius: '8px',
          overflow: 'hidden',
        }}
      >
        <Canvas
          camera={{ position: [5, 5, 5], fov: 50 }}
          gl={{ antialias: true }}
        >
          {/* Lighting */}
          <ambientLight intensity={0.5} />
          <directionalLight position={[10, 10, 5]} intensity={1} />

          {/* Camera controls (rotate, zoom, pan) */}
          <OrbitControls />

          {/* Environment (reflections) */}
          <Environment preset="studio" />

          {/* Grid helper */}
          <Grid infiniteGrid />

          {/* The actual geometry */}
          {shape.props.meshData && (
            <GeometryMesh meshData={shape.props.meshData} />
          )}
        </Canvas>

        {/* Info overlay */}
        <div
          style={{
            position: 'absolute',
            top: 8,
            left: 8,
            background: 'rgba(0, 0, 0, 0.7)',
            color: 'white',
            padding: '8px 12px',
            borderRadius: '4px',
            fontSize: '12px',
            fontFamily: 'monospace',
          }}
        >
          {shape.props.meshData
            ? `${shape.props.meshData.vertices.length / 3} vertices`
            : 'No geometry'}
        </div>
      </div>
    </HTMLContainer>
  )
}

// Component that renders the mesh
function GeometryMesh({ meshData }: { meshData: MeshData }) {
  const geometry = new THREE.BufferGeometry()

  // Set vertex positions
  geometry.setAttribute(
    'position',
    new THREE.BufferAttribute(meshData.vertices, 3)
  )

  // Set normals (for lighting)
  geometry.setAttribute(
    'normal',
    new THREE.BufferAttribute(meshData.normals, 3)
  )

  // Set indices (triangle connectivity)
  geometry.setIndex(new THREE.BufferAttribute(meshData.indices, 1))

  return (
    <mesh geometry={geometry}>
      <meshStandardMaterial
        color="#4a9eff"
        metalness={0.3}
        roughness={0.4}
      />
    </mesh>
  )
}
```

Register the shape:

```typescript
// src/components/TldrawKclCanvas.tsx
import { GeometryViewerShapeUtil } from '../shapes/GeometryViewerShape'

const customShapeUtils = [
  KclCodeShapeUtil,
  GeometryViewerShapeUtil,  // Add this
]
```

## Converting KCL Geometry to Three.js Format

KCL's geometry engine returns geometry in a specific format. You need to convert it to Three.js format (vertices, normals, indices).

Create `src/lib/geometryConversion.ts`:

```typescript
import { MeshData } from '../shapes/GeometryViewerShape'

export interface KclGeometry {
  // This structure depends on your engine's output
  vertices: number[]      // [x, y, z, x, y, z, ...]
  faces: number[][]       // [[v1, v2, v3], [v4, v5, v6], ...]
  normals?: number[]      // [nx, ny, nz, nx, ny, nz, ...]
}

export function convertKclToMeshData(kclGeom: KclGeometry): MeshData {
  const vertices = new Float32Array(kclGeom.vertices)

  // Convert face indices to flat array
  const indices = new Uint32Array(
    kclGeom.faces.flat()
  )

  // Calculate normals if not provided
  const normals = kclGeom.normals
    ? new Float32Array(kclGeom.normals)
    : calculateNormals(vertices, indices)

  return { vertices, indices, normals }
}

function calculateNormals(
  vertices: Float32Array,
  indices: Uint32Array
): Float32Array {
  const normals = new Float32Array(vertices.length)

  // For each triangle
  for (let i = 0; i < indices.length; i += 3) {
    const i1 = indices[i] * 3
    const i2 = indices[i + 1] * 3
    const i3 = indices[i + 2] * 3

    // Get triangle vertices
    const v1 = [vertices[i1], vertices[i1 + 1], vertices[i1 + 2]]
    const v2 = [vertices[i2], vertices[i2 + 1], vertices[i2 + 2]]
    const v3 = [vertices[i3], vertices[i3 + 1], vertices[i3 + 2]]

    // Calculate normal via cross product
    const edge1 = [v2[0] - v1[0], v2[1] - v1[1], v2[2] - v1[2]]
    const edge2 = [v3[0] - v1[0], v3[1] - v1[1], v3[2] - v1[2]]

    const normal = [
      edge1[1] * edge2[2] - edge1[2] * edge2[1],
      edge1[2] * edge2[0] - edge1[0] * edge2[2],
      edge1[0] * edge2[1] - edge1[1] * edge2[0],
    ]

    // Normalize
    const length = Math.sqrt(
      normal[0] ** 2 + normal[1] ** 2 + normal[2] ** 2
    )
    normal[0] /= length
    normal[1] /= length
    normal[2] /= length

    // Add to each vertex of the triangle
    for (const idx of [i1, i2, i3]) {
      normals[idx] += normal[0]
      normals[idx + 1] += normal[1]
      normals[idx + 2] += normal[2]
    }
  }

  // Normalize all normals
  for (let i = 0; i < normals.length; i += 3) {
    const length = Math.sqrt(
      normals[i] ** 2 + normals[i + 1] ** 2 + normals[i + 2] ** 2
    )
    normals[i] /= length
    normals[i + 1] /= length
    normals[i + 2] /= length
  }

  return normals
}
```

## Linking Code to Geometry

Now connect KCL code shapes to geometry viewer shapes. Update the KCL execution:

```typescript
// src/lib/kclExecution.ts

export async function executeKcl(code: string): Promise<{
  output: string
  variables: Record<string, any>
  geometry: KclGeometry | null
  errors: string[]
}> {
  await ensureWasmInitialized()

  try {
    const result = execute_kcl(code)

    return {
      output: formatOutput(result),
      variables: result.variables || {},
      geometry: result.geometry || null,  // Add geometry
      errors: result.errors || [],
    }
  } catch (e) {
    return {
      output: '',
      variables: {},
      geometry: null,
      errors: [e.message],
    }
  }
}
```

Update the KCL code shape to create/update a geometry viewer:

```typescript
// src/shapes/KclCodeShape.tsx
import { convertKclToMeshData } from '../lib/geometryConversion'
import { GeometryViewerShape } from './GeometryViewerShape'

function KclCodeEditor({ shape }: { shape: KclCodeShape }) {
  const editor = useEditor()
  const [linkedViewerId, setLinkedViewerId] = useState<string | null>(null)

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

          // Update or create geometry viewer
          if (result.geometry) {
            updateGeometryViewer(result.geometry)
          }
        }
      } catch (e) {
        setError(e.message)
        setOutput('')
      }
    }, 500)
  ).current

  const updateGeometryViewer = (geometry: KclGeometry) => {
    const meshData = convertKclToMeshData(geometry)

    if (linkedViewerId) {
      // Update existing viewer
      const viewer = editor.getShape<GeometryViewerShape>(linkedViewerId)
      if (viewer) {
        editor.updateShape<GeometryViewerShape>({
          id: linkedViewerId,
          type: 'geometry-viewer',
          props: {
            ...viewer.props,
            meshData,
          },
        })
      }
    } else {
      // Create new viewer to the right of the code shape
      const newViewer = editor.createShape<GeometryViewerShape>({
        type: 'geometry-viewer',
        x: shape.x + shape.props.w + 20,
        y: shape.y,
        props: {
          w: 400,
          h: 400,
          geometryId: shape.id,
          meshData,
        },
      })

      setLinkedViewerId(newViewer.id)

      // Store the link in the code shape props
      editor.updateShape<KclCodeShape>({
        id: shape.id,
        type: 'kcl-code',
        props: {
          ...shape.props,
          linkedViewerId: newViewer.id,
        },
      })
    }
  }

  // ... rest of component
}
```

Update the shape type to store the link:

```typescript
export type KclCodeShape = TLBaseShape<
  'kcl-code',
  {
    w: number
    h: number
    code: string
    linkedViewerId: string | null  // Add this
  }
>
```

## Visual Linking (Arrows Between Shapes)

Show a visual connection between code and geometry:

```typescript
// src/components/ShapeLinks.tsx
import { useEditor } from '@tldraw/tldraw'
import { useEffect, useState } from 'react'

export function ShapeLinks() {
  const editor = useEditor()
  const [links, setLinks] = useState<Array<{ from: string; to: string }>>([])

  useEffect(() => {
    const updateLinks = () => {
      const allShapes = editor.getCurrentPageShapes()
      const codeShapes = allShapes.filter(s => s.type === 'kcl-code')

      const newLinks = codeShapes
        .filter(s => s.props.linkedViewerId)
        .map(s => ({
          from: s.id,
          to: s.props.linkedViewerId,
        }))

      setLinks(newLinks)
    }

    updateLinks()

    // Listen for shape changes
    editor.store.listen(() => {
      updateLinks()
    })
  }, [editor])

  return (
    <svg
      style={{
        position: 'absolute',
        inset: 0,
        pointerEvents: 'none',
        zIndex: 1000,
      }}
    >
      {links.map(link => {
        const fromShape = editor.getShape(link.from)
        const toShape = editor.getShape(link.to)

        if (!fromShape || !toShape) return null

        const fromCenter = {
          x: fromShape.x + fromShape.props.w / 2,
          y: fromShape.y + fromShape.props.h / 2,
        }

        const toCenter = {
          x: toShape.x + toShape.props.w / 2,
          y: toShape.y + toShape.props.h / 2,
        }

        return (
          <line
            key={link.from}
            x1={fromCenter.x}
            y1={fromCenter.y}
            x2={toCenter.x}
            y2={toCenter.y}
            stroke="#4a9eff"
            strokeWidth={2}
            strokeDasharray="5,5"
            opacity={0.5}
          />
        )
      })}
    </svg>
  )
}
```

Add it to your canvas:

```typescript
// src/components/TldrawKclCanvas.tsx
export function TldrawKclCanvas() {
  return (
    <div style={{ position: 'fixed', inset: 0 }}>
      <Tldraw shapeUtils={customShapeUtils} tools={customTools}>
        <ShapeLinks />
      </Tldraw>
    </div>
  )
}
```

## Example: Box to 3D

Test the full pipeline. Create a KCL code shape and type:

```kcl
width = 100
height = 50
depth = 25

box = startSketchOn("XY")
  |> startProfile(at = [0, 0])
  |> line(to = [width, 0])
  |> line(to = [width, height])
  |> line(to = [0, height])
  |> close()
  |> extrude(depth)
```

As you type, a geometry viewer appears to the right, showing a 3D box. Change `width` to 200, the box updates immediately. Rotate the view with your mouse.

## Adding Export Options

Let users export geometry:

```typescript
// src/components/GeometryExport.tsx
import { MeshData } from '../shapes/GeometryViewerShape'

export function exportToSTL(meshData: MeshData, filename: string) {
  const stl = generateSTL(meshData)
  const blob = new Blob([stl], { type: 'application/sla' })
  const url = URL.createObjectURL(blob)

  const a = document.createElement('a')
  a.href = url
  a.download = filename
  a.click()

  URL.revokeObjectURL(url)
}

function generateSTL(meshData: MeshData): string {
  let stl = 'solid model\n'

  for (let i = 0; i < meshData.indices.length; i += 3) {
    const i1 = meshData.indices[i] * 3
    const i2 = meshData.indices[i + 1] * 3
    const i3 = meshData.indices[i + 2] * 3

    const v1 = [
      meshData.vertices[i1],
      meshData.vertices[i1 + 1],
      meshData.vertices[i1 + 2],
    ]
    const v2 = [
      meshData.vertices[i2],
      meshData.vertices[i2 + 1],
      meshData.vertices[i2 + 2],
    ]
    const v3 = [
      meshData.vertices[i3],
      meshData.vertices[i3 + 1],
      meshData.vertices[i3 + 2],
    ]

    const n1 = [
      meshData.normals[i1],
      meshData.normals[i1 + 1],
      meshData.normals[i1 + 2],
    ]

    stl += `  facet normal ${n1[0]} ${n1[1]} ${n1[2]}\n`
    stl += `    outer loop\n`
    stl += `      vertex ${v1[0]} ${v1[1]} ${v1[2]}\n`
    stl += `      vertex ${v2[0]} ${v2[1]} ${v2[2]}\n`
    stl += `      vertex ${v3[0]} ${v3[1]} ${v3[2]}\n`
    stl += `    endloop\n`
    stl += `  endfacet\n`
  }

  stl += 'endsolid model\n'
  return stl
}
```

Add export button to geometry viewer:

```typescript
function GeometryViewer({ shape }: { shape: GeometryViewerShape }) {
  const handleExport = () => {
    if (shape.props.meshData) {
      exportToSTL(shape.props.meshData, 'model.stl')
    }
  }

  return (
    <HTMLContainer>
      <div style={{ /* ... */ }}>
        <Canvas>{/* ... */}</Canvas>

        {/* Export button */}
        <button
          onClick={handleExport}
          style={{
            position: 'absolute',
            bottom: 8,
            right: 8,
            padding: '8px 12px',
            background: '#007acc',
            color: 'white',
            border: 'none',
            borderRadius: '4px',
            cursor: 'pointer',
          }}
        >
          Export STL
        </button>
      </div>
    </HTMLContainer>
  )
}
```

## Multiple Views of the Same Model

Create multiple geometry viewers linked to the same code, showing different angles:

```typescript
// Add a "Add View" button to the KCL code shape
const addView = () => {
  if (lastGeometry) {
    const meshData = convertKclToMeshData(lastGeometry)

    const newViewer = editor.createShape<GeometryViewerShape>({
      type: 'geometry-viewer',
      x: shape.x + shape.props.w + 20,
      y: shape.y + 420, // Below existing viewer
      props: {
        w: 400,
        h: 400,
        geometryId: shape.id,
        meshData,
      },
    })
  }
}
```

Now you can have top view, front view, side view, and isometric view all on the same canvas.

## What You Just Built

- Interactive 3D geometry viewer as a tldraw shape
- Real-time updates (code changes → geometry updates)
- Visual links between code and geometry
- Multiple views of the same model
- STL export functionality
- Proper lighting, materials, and camera controls

## What You Just Learned

1. Three.js basics (scene, camera, renderer, mesh)
2. React Three Fiber (declarative Three.js)
3. Buffer geometries (vertices, normals, indices)
4. Normal calculation for lighting
5. Linking tldraw shapes together
6. Real-time data flow between shapes
7. STL file format and export

## Exercises

1. **Add camera presets**: Buttons for top view, front view, side view, isometric view.

2. **Add measurement tools**: Click two points, show the distance.

3. **Add cross-sections**: Slice the geometry with a plane, show the cross-section.

4. **Add material editor**: Let users change color, metalness, roughness interactively.

5. **Add animation**: Animate a parameter (e.g., slowly increase extrusion depth) and watch the model grow.

## Next Up

In Part 13, we'll add AI-powered sketch-to-KCL conversion. You'll draw rough shapes with tldraw's draw tool, and AI will convert them to precise KCL code. makereal meets CAD. This is the final piece: low-fidelity sketches, AI conversion, high-fidelity code, and 3D geometry, all on one infinite canvas.

Time to close the loop.
