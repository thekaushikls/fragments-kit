# Working with Fragments (.frag) FlatBuffers

## Overview

`.frag` files are FlatBuffers binary files (file identifier `"0001"`) that store BIM model data. The schema is defined in `fragments/Fragments.fbs` and the generated Python bindings are in `fragments/fragments.py`.

Files may be zlib-compressed (headers: `78 9c`, `78 01`, `78 da`). Decompress before parsing.

```python
from fragments.fragments import Model
root: Model = Model.GetRootAs(buf, 0)
```

---

## Data Hierarchy

```
Model (root)
├── metadata: string (JSON)
├── guid: string (unique model ID)
├── max_local_id: uint
│
├── local_ids: [uint]              # File-specific item IDs (parallel array)
├── categories: [string]           # IFC category per item (parallel array)
├── attributes: [Attribute]        # Item data per item (parallel array)
│   └── data: [string]             # Each entry is JSON: ["Name","value","IFCTYPE"]
│
├── guids: [string]                # Global Unique Identifiers (not all items have one)
├── guids_items: [uint]            # Maps localIds indices -> guids
│
├── relations: [Relation]          # Relations between items
│   └── data: [string]
├── relations_items: [int]         # Maps localIds indices -> relations
│
├── unique_attributes: [string]    # Unique attribute names
├── relation_names: [string]       # Unique relation names
│
├── spatial_structure: SpatialStructure  # Tree of spatial relations
│   ├── local_id: uint
│   ├── category: string           # e.g. "IFCPROJECT", "IFCSITE", "IFCBUILDING", "IFCBUILDINGSTOREY"
│   └── children: [SpatialStructure]
│
└── meshes: Meshes                 # All geometry
    ├── coordinates: Transform     # Global model coordinates (geographic location)
    ├── shells: [Shell]            # B-rep geometry (faces + holes)
    ├── circle_extrusions: [CircleExtrusion]  # Wire-with-thickness (rebar)
    ├── representations: [Representation]     # Common geometry interface
    ├── samples: [Sample]          # Geometry instances
    ├── materials: [Material]      # RGBA + render properties
    ├── local_transforms: [Transform]
    ├── global_transforms: [Transform]
    ├── meshes_items: [uint]       # Maps localIds indices -> global transforms
    ├── material_ids: [uint]       # Local IDs for materials
    ├── representation_ids: [uint] # Local IDs for representations
    ├── sample_ids: [uint]         # Local IDs for samples
    ├── local_transform_ids: [uint]
    └── global_transform_ids: [uint]
```

### Parallel Arrays

`local_ids`, `categories`, and `attributes` are parallel arrays indexed the same way. Index `i` across all three refers to the same item:

```python
local_id = root.LocalIds(i)
category = root.Categories(i)     # e.g. b"IFCWALLSTANDARDCASE"
attribute = root.Attributes(i)    # Attribute table with .Data(j)
```

---

## Spatial Structure

A recursive tree representing IFC spatial hierarchy. The pattern alternates between:
- **Category nodes**: `category` is set (e.g. `b"IFCBUILDINGSTOREY"`), `local_id` is `None`
- **Data nodes**: `local_id` is set, `category` is `None`, followed by children that are the items belonging to this spatial element

```
IFCPROJECT (category node)
  └── local_id=120 (data node)
        └── IFCSITE (category node)
              └── local_id=208 (data node)
                    └── IFCBUILDING (category node)
                          └── local_id=130 (data node)
                                ├── IFCBUILDINGSTOREY (category node)
                                │     └── local_id=139 (data node)
                                │           ├── IFCWALLSTANDARDCASE ...
                                │           └── IFCSLAB ...
                                └── IFCBUILDINGSTOREY (category node)
                                      └── local_id=145 (data node)
                                            └── ...
```

To find items of a specific IFC type in the tree:
```python
def walk(node, target_category):
    if node is None:
        return []
    results = []
    if node.Category() == target_category:
        for i in range(node.ChildrenLength()):
            child = node.Children(i)
            if child.LocalId() is not None and child.Category() is None:
                results.append(child.LocalId())
    for i in range(node.ChildrenLength()):
        results.extend(walk(node.Children(i), target_category))
    return results
```

---

## Attributes

Each `Attribute.Data(j)` is a JSON-encoded string with the format:

```json
["PropertyName", value, "IFCTYPE"]
```

Examples:
```
["Name", "Floor 0", "IFCLABEL"]
["ObjectType", "Level:Elevationsmarkering", "IFCLABEL"]
["Elevation", 0, "IFCLENGTHMEASURE"]
```

To extract a named property:
```python
attr = root.Attributes(idx)
for j in range(attr.DataLength()):
    entry = json.loads(attr.Data(j))
    if entry[0] == "Name":
        name = entry[1]
```

---

## Geometry: Shells (B-rep)

Shells are the primary geometry type. Each shell stores faces as polygon loops referencing shared vertices.

### Vertices
```python
shell = meshes.Shells(i)
for pt_idx in range(shell.PointsLength()):
    pt = shell.Points(pt_idx)
    x, y, z = pt.X(), pt.Y(), pt.Z()
```

### Face Profiles (polygon loops)
Each `ShellProfile` contains indices into the shell's `points` array forming a closed polygon:

```python
for p_idx in range(shell.ProfilesLength()):
    profile = shell.Profiles(p_idx)
    indices = [profile.Indices(i) for i in range(profile.IndicesLength())]
    # indices form a closed polygon referencing shell.Points()
```

### Holes
`ShellHole` is like `ShellProfile` but represents interior cutouts. Each hole has a `profile_id` linking it to its parent profile:

```python
for h_idx in range(shell.HolesLength()):
    hole = shell.Holes(h_idx)
    parent_profile = hole.ProfileId()
    indices = [hole.Indices(i) for i in range(hole.IndicesLength())]
```

### Big Shells
When a shell has more than 65535 points, `shell.Type() == ShellType.BIG` and you must use `BigProfiles`/`BigHoles` (uint32 indices) instead of `Profiles`/`Holes` (uint16 indices).

### Triangulation
Profiles are polygon loops, not triangles. To get triangles (e.g. for dotbim or GPU rendering), use fan triangulation:

```python
# For a polygon with indices [a, b, c, d, e]:
# Triangles: (a,b,c), (a,c,d), (a,d,e)
for i in range(1, len(indices) - 1):
    tri = (indices[0], indices[i], indices[i + 1])
```

Fan triangulation works for convex polygons. For concave polygons, use ear-clipping.

---

## Geometry: Circle Extrusions

Used for reinforcement bars. A wire path with a circular cross-section:

```python
ce = meshes.CircleExtrusions(i)
for j in range(ce.RadiusLength()):
    radius = ce.Radius(j)        # half-thickness
for j in range(ce.AxesLength()):
    axis = ce.Axes(j)
    # axis.Wires(), axis.WireSets(), axis.CircleCurves()
    # axis.Order() and axis.Parts() define the sequence
```

---

## Samples (Geometry Instances)

A `Sample` ties geometry to its placement, material, and transform:

```python
sample = meshes.Samples(i)
sample.Item()            # -> index into meshes_items -> global_transforms
sample.Material()        # -> index into materials
sample.Representation()  # -> index into representations
sample.LocalTransform()  # -> index into local_transforms
```

### Representation -> Geometry lookup
```python
rep = meshes.Representations(sample.Representation())
rep.Id()                    # index into shells[] or circle_extrusions[]
rep.RepresentationClass()   # SHELL (1) or CIRCLE_EXTRUSION (2)
rep.Bbox()                  # BoundingBox with .Min() and .Max()
```

### Material
```python
mat = meshes.Materials(sample.Material())
r, g, b, a = mat.R(), mat.G(), mat.B(), mat.A()
mat.RenderedFaces()  # ONE (0) or TWO (1) - single or double-sided
mat.Stroke()         # line stroke type
```

---

## Transforms

```python
tf = meshes.LocalTransforms(i)   # or GlobalTransforms(i)
tf.Position()      # DoubleVector: .X(), .Y(), .Z()
tf.XDirection()    # FloatVector: .X(), .Y(), .Z()
tf.YDirection()    # FloatVector: .X(), .Y(), .Z()
# Z direction = cross(X, Y)
```

To build a 4x4 matrix:
```python
x_dir = [tf.XDirection().X(), tf.XDirection().Y(), tf.XDirection().Z()]
y_dir = [tf.YDirection().X(), tf.YDirection().Y(), tf.YDirection().Z()]
z_dir = np.cross(x_dir, y_dir)
pos   = [tf.Position().X(), tf.Position().Y(), tf.Position().Z()]

mat4 = np.eye(4)
mat4[:3, 0] = x_dir
mat4[:3, 1] = y_dir
mat4[:3, 2] = z_dir
mat4[:3, 3] = pos
```

World position of a sample: `global_transform @ local_transform @ vertex`

---

## Performance: NumPy Accessors

Scalar vector fields have `.AsNumpy()` methods that return zero-copy numpy arrays, much faster than Python loops:

```python
root.LocalIdsAsNumpy()                  # np.ndarray of uint32
meshes.MeshesItemsAsNumpy()             # np.ndarray of uint32
meshes.MaterialIdsAsNumpy()             # np.ndarray of uint32
meshes.RepresentationIdsAsNumpy()       # np.ndarray of uint32
meshes.SampleIdsAsNumpy()               # np.ndarray of uint32
meshes.LocalTransformIdsAsNumpy()       # np.ndarray of uint32
meshes.GlobalTransformIdsAsNumpy()      # np.ndarray of uint32
root.GuidsItemsAsNumpy()                # np.ndarray of uint32
shell.ProfilesFaceIdsAsNumpy()          # np.ndarray of uint16
shell_profile.IndicesAsNumpy()          # np.ndarray of uint16
big_shell_profile.IndicesAsNumpy()      # np.ndarray of uint32
circle_extrusion.RadiusAsNumpy()        # np.ndarray of float64
axis.OrderAsNumpy()                     # np.ndarray of uint32
axis.PartsAsNumpy()                     # np.ndarray of int8
```

**Not available** for string vectors (`Categories`, `Guids`) or table vectors (`Shells`, `Attributes`, `Samples`, etc.).

Use `np.isin` / `np.where` for vectorized lookups instead of Python loops:
```python
local_ids = root.LocalIdsAsNumpy()
indices = np.where(np.isin(local_ids, target_ids))[0]
```

---

## IFC Category Reference

Common categories found in .frag files (from BasicHouse.frag):

| Category | Description |
|---|---|
| `IFCPROJECT` | Top-level project |
| `IFCSITE` | Site |
| `IFCBUILDING` | Building |
| `IFCBUILDINGSTOREY` | Floor/storey |
| `IFCWALLSTANDARDCASE` | Walls |
| `IFCSLAB` | Floor/roof slabs |
| `IFCDOOR` | Doors |
| `IFCWINDOW` | Windows |
| `IFCFURNISHINGELEMENT` | Furniture |
| `IFCBUILDINGELEMENTPROXY` | Generic elements |
| `IFCMEMBER` | Structural members |
| `IFCPLATE` | Plates |
| `IFCRAILING` | Railings |
| `IFCSTAIRFLIGHT` | Stair flights |
