# Pygccx codebase guide

Pygccx is a Python framework for building, solving, and post-processing
CalculiX finite-element models. The normal workflow is:

```text
geometry/mesh source -> pygccx.Mesh -> Model + keyword objects
                     -> jobname.inp -> CalculiX
                     -> jobname.frd / jobname.dat
                     -> result readers -> NumPy/post-processing
```

This guide explains where each responsibility lives and how the main objects
relate. The companion [relational map](relational-map.md) provides the
component and data-flow view.

## Repository layout

| Path | Responsibility |
| --- | --- |
| `pygccx/model.py` | `Model`, the orchestration object for mesh, keywords, steps, external programs, and results |
| `pygccx/mesh/` | Nodes, elements, sets, surfaces, element topology, and CalculiX mesh serialization |
| `pygccx/mesh/mesh_factory/` | Conversion from Gmsh, CalculiX `.inp`, and CGX/CalculiX `.frd` meshes |
| `pygccx/model_keywords/` | Model-level CalculiX keyword objects such as materials, sections, constraints, contact, and coupling |
| `pygccx/step_keywords/` | Analysis-step objects and load/output keyword objects |
| `pygccx/result_reader/` | Parsers and query objects for ASCII `.frd` and `.dat` results |
| `pygccx/helper_features/` | Reusable modeling helpers, currently coordinate-system transformations |
| `pygccx/tools/` | Higher-level bolt generation/post-processing and stress calculations |
| `pygccx/enums.py` | Element, set, surface, analysis, load, and result enumerations |
| `pygccx/protocols.py` | Structural interfaces used to keep mesh and keyword components loosely coupled |
| `pygccx/auxiliary.py` | Shared formatting helpers used while writing input syntax |
| `pygccx/exceptions.py` | Domain-specific errors, especially unsupported element handling |
| `examples/` | End-to-end modeling and post-processing scripts |
| `tests/` | Unit and integration-style coverage grouped by subsystem |
| `docs/pygccx/` | Generated API documentation; source code is the authoritative reference |

The package is configured for Python 3.10+ and depends on NumPy, SciPy, and
the Gmsh Python API. CalculiX `ccx` and `cgx` are external executables used by
`Model.solve()` and the CGX display methods.

## Core object model

### `Model`: the workflow root

Instantiate `pygccx.model.Model` with paths to `ccx` and `cgx`, plus an
optional job name and working directory:

```python
from pygccx import model as ccx_model

model = ccx_model.Model("ccx", "cgx", jobname="beam", working_dir="results")
```

`Model.__post_init__` creates a dedicated Gmsh model and an initially empty
`Mesh`. The model owns:

- `mesh`: the current `pygccx.mesh.Mesh`;
- `model_keywords`: objects written before the analysis steps;
- `steps`: `Step` objects, each containing step keyword objects;
- solver/display paths and generated-file metadata.

The important methods form the lifecycle:

1. `get_gmsh()` returns the Gmsh API with this model selected.
2. `update_mesh_from_gmsh()`, `update_mesh_from_inp()`, or
   `update_mesh_from_frd()` replaces `model.mesh` with a converted mesh.
3. `add_model_keywords()` appends model-level keyword objects.
4. `add_steps()` appends analysis steps.
5. `write_ccx_input_file()` serializes mesh, model keywords, and steps to
   `jobname.inp`.
6. `solve()` optionally writes the input and runs `ccx -i jobname`.
7. `get_frd_result()` and `get_dat_result()` parse solver output.
8. `to_pickle()` / `Model.from_pickle()` preserve the Python model for later
   post-processing. The Gmsh model itself is intentionally not serialized.

Use the model as a context manager when possible. Its `__exit__` removes the
associated Gmsh model.

### `Mesh`: the finite-element data graph

`Mesh` stores:

- `nodes`: `{node_id: (x, y, z)}`;
- `elements`: `{element_id: Element}`;
- `node_sets` and `element_sets`: named collections of IDs;
- `surfaces`: node-based or element-face-based interfaces.

`Element` references node IDs and has an `EEtypes` type. Its topology helpers
(`get_dim()`, corner-node methods, and `get_faces()`) are used by surface
creation and higher-level tools. `Set` and the surface classes provide the
named entities consumed by keyword objects.

Mesh serialization is deliberately simple and deterministic in structure:

```text
*NODE
*ELEMENT,TYPE=...
*NSET / *ELSET
*SURFACE
```

`Mesh.write_ccx(buffer)` emits those sections. It does not write model
materials, loads, constraints, or analysis steps; those belong to keyword
objects and `Model`.

Mesh mutation is supported through methods such as `add_node`,
`add_element`, `add_set`, `add_node_surface`, and
`add_el_face_surface`. IDs are generated from the current maximum when
omitted. Names are generally normalized to uppercase when stored.

### Mesh factories and source boundaries

The factory package translates external representations into the internal
`Mesh` graph:

- `gmsh_factory.mesh_from_gmsh(gmsh, ...)`
  - reads Gmsh nodes and 3D elements;
  - maps Gmsh element numbers to `EEtypes`;
  - maps Gmsh physical groups to node and element sets;
  - reorders higher-order nodes where CalculiX expects a different order.
- `inp_factory.mesh_from_inp(filename, ...)`
  - reads nodes, supported solid elements, node/element sets, and surfaces;
  - expands `*INCLUDE` recursively;
  - can skip unsupported element blocks;
  - can clear unused IDs from sets and surfaces.
- `frd_factory.mesh_from_frd(filename, ...)`
  - converts supported solid mesh data from an FRD file.

The supported mesh scope is primarily 3D solid elements. Abstract elements
such as springs, gaps, dashpots, masses, and couplings can be added directly
to a `Mesh`, but unsupported shell/beam workflows should not be assumed to
work through the factories.

## Keyword serialization

Keyword classes are dataclasses with a `__str__()` method that returns the
CalculiX input fragment they represent. Most classes also expose `name` and
`desc`, matching `IKeyword`.

### Model keywords

`pygccx.model_keywords` contains definitions that occur outside a step:

- model metadata: `Heading`, `Include`, `Universal`;
- materials and sections: `Material`, `Elastic`, `HyperElastic`, `Density`,
  `Plastic`, `SolidSection`, and related classes;
- constraints and loads attached to the model: `Boundary`, `Equation`,
  `Mpc`, `RigidBody`, `Coupling`, `DistributingCoupling`;
- contact and interfaces: `Friction`, `SurfaceBehavior`,
  `SurfaceInteraction`, `ContactPair`, `Clearance`, `Tie`, `Gap`;
- coordinates and special features: `Orientation`, `Transform`,
  `PretensionSection`, `CyclicSymmetryModel`, and others.

Many model keywords reference mesh entities by protocol rather than by a
concrete class. For example, `SolidSection` consumes an element set and a
`Material`; `Boundary` consumes a node set or node; contact keywords consume
surfaces. The references are converted to names/IDs while serializing.

`Universal` is the escape hatch for an input fragment not yet represented by a
dedicated class. Prefer a dedicated keyword when one exists because dedicated
classes validate their inputs and compose more clearly.

### Steps and step keywords

`Step` represents `*STEP` and owns an ordered `step_keywords` list. Its
configuration controls `NLGEOM`, increment limits, amplitude behavior, and
perturbation mode. Step keyword classes describe the analysis and its
contents, for example:

- analysis: `Static`, `Visco`, `Buckle`, `Frequency`, `Green`, `NoAnalysis`;
- loads/conditions: `Boundary`, `Cload`, `Dload`;
- output: `NodeFile`, `ElFile`, `NodePrint`, `ElPrint`,
  `NodeOutput`, `ElementOutput`, and contact output classes;
- step support: `TimePoints`, `ContactFile`, `ContactPrint`.

`Model.write_ccx_input_file()` writes each step in order, writes each
step-keyword string in order, and appends `*END STEP`. Ordering therefore
matters: add keywords in the order expected by the CalculiX input format.

## End-to-end workflow

### 1. Build or import geometry

Use `model.get_gmsh()` for geometry and meshing with Gmsh, or start with an
existing `.inp`/`.frd` file. Define Gmsh physical groups when using Gmsh;
those groups become named sets during conversion.

### 2. Convert into `Mesh`

Call the matching `update_mesh_from_*` method. After conversion, work with
`model.mesh` rather than the Gmsh representation. Important: each update
replaces the mesh object, so references to an older mesh, set, or surface
must not be reused after another update.

### 3. Add model entities

Create or retrieve sets and surfaces, then add materials, sections,
constraints, couplings, contact, and other model keywords. A common pattern
is:

```python
mesh = model.mesh
material = mk.Material("STEEL")
elastic = mk.Elastic((210000.0, 0.3))
section = mk.SolidSection(mesh.get_el_set_by_name("PART"), material)
model.add_model_keywords(material, elastic, section)
```

### 4. Add analysis steps and requested outputs

Create a `Step`, add an analysis keyword and loads, then request the result
entities needed later. FRD readers require ASCII output generated by
`*NODE FILE`, `*EL FILE`, and `*CONTACT FILE`; DAT readers consume print
output such as `*NODE PRINT`, `*EL PRINT`, and contact print output.

### 5. Write and solve

`write_ccx_input_file()` is useful for inspection and debugging. `solve()`
executes the external solver in `working_dir`; it does not replace the need
to validate that the configured executable paths and input are correct.

### 6. Read and post-process

`FrdResult` and `DatResult` are immutable result containers with
`get_result_sets_by()` and `get_available_times()` queries. Their result-set
values are NumPy arrays keyed by node or element ID. `DatResult` also exposes
step metadata for buckling and frequency analyses.

Use `pygccx.tools.stress_tools` for tensor invariants and
`helper_features.CoordinateSystem` for coordinate transformations. The
`Bolt` tool builds a reusable bolt submodel, inserts generated mesh and
keywords into a `Model`, and can derive section forces from FRD reaction
forces.

## Extension guide

### Adding a model or step keyword

1. Add a dataclass in the appropriate package.
2. Give it a `name` and `desc` field when it participates in the keyword
   protocol.
3. Validate references and mutually exclusive options in `__setattr__`,
   `__post_init__`, or a local validation helper.
4. Implement `__str__()` to return only the corresponding CalculiX fragment.
5. Export the class from the package `__init__.py`.
6. Add focused tests under `tests/tests_model_keywords` or
   `tests/tests_step_keywords`, plus an example if the interaction is
   non-obvious.

### Adding a mesh element or source conversion

Update the relevant enum and topology tables first. Keep node ordering
consistent with CalculiX, add conversion mappings in the factory, and test
serialization plus round-tripping where practical. Unsupported input should
raise `ElementTypeNotSupportedError` unless the caller explicitly requests
skipping.

### Adding a result entity

Update the relevant enum, output keyword support, parser mapping, and result
location semantics together. Test both parsing and `get_values_by_ids()`;
the latter is the main bridge from parsed results to user post-processing.

## Testing and navigation

Run the complete suite with:

```bash
python -m unittest tests.test_all
```

Subsystem entry points are also available through:

```bash
python -m unittest tests.test_mesh
python -m unittest tests.test_model_keywords
python -m unittest tests.test_step_keywords
python -m unittest tests.test_result_reader
python -m unittest tests.test_stress_tools
python -m unittest tests.test_bolts
```

When investigating a behavior, start at the public method in `model.py`,
follow the protocol type into the mesh or keyword package, then inspect the
corresponding test and example. Generated HTML under `docs/pygccx/` is useful
for API browsing, but it can lag behind source changes.

