# 🖨️ NovaMake Worker: Slicer Check

> **The Gatekeeper of Distributed Manufacturing.**
> Automated geometric analysis, printability verification, and G-code generation for the NovaMake grid.

[](https://www.google.com/search?q=https://github.com/novaeco-tech/novamake-worker-slicer-check/actions)
[](https://opensource.org/licenses/MIT)
[](https://www.google.com/search?q=https://slicer.make.novaeco.tech)

**SlicerCheck** is a headless, compute-intensive worker belonging to the **[NovaMake](https://www.google.com/search?q=https://manufacturing.novaeco.tech)** sector.

In a distributed manufacturing network, we cannot rely on every FabLab operator to manually check every customer file. This worker automates the "Pre-Flight Check." It consumes raw 3D models (STL/OBJ), verifies their geometry, calculates material requirements, and generates machine-specific instructions (G-code).

-----

## 🎯 Value Proposition

3D printing waste is a major circular economy issue. "Spaghetti prints" (failed jobs) waste energy and plastic. This worker solves this by:

1.  **Waste Prevention:** Detecting non-manifold edges, inverted normals, or impossible overhangs *before* the job is sent to a printer.
2.  **Accurate Costing:** Precisely calculating filament usage (in grams) and print time (in minutes) to generate fair quotes in **NovaTrade**.
3.  **Material Impact:** sending the exact material mass to **Worker-LCA** to calculate the carbon footprint of the specific object.

-----

## 🏗️ Architecture (The Analysis Pipeline)

This worker consumes the `queue.make.slicer-jobs` queue managed by NovaMake.

```mermaid
graph LR
    subgraph "Fabrication Request (NovaMake)"
        Make[Job Manager] -->|1. New Model Uploaded| MQ[(RabbitMQ)]
    end

    subgraph "The Worker (NovaMake)"
        MQ -->|2. Fetch STL| Worker[Slicer Worker]
        Worker -->|3. Analyze Geometry| Mind["NovaMind (Geometry Engine)"]
        Worker -->|4. Generate G-Code| Cura[Headless CuraEngine]
    end

    subgraph "Outputs"
        Worker -->|5. Print Time & Grams| Make
        Worker -->|6. Impact Data| Balance[NovaBalance LCA]
    end
```

### The Slicing Lifecycle

1.  **Ingest:** Receives a job containing the file URL (S3) and the target Printer Profile (e.g., "Prusa Mk3S / PETG").
2.  **Mesh Analysis:** Checks if the model is "Watertight" (manifold). If holes are found, it attempts a repair via `NovaMind` or rejects the job.
3.  **Slicing:** Runs a headless slicing engine to generate layers and toolpaths.
4.  **Estimation:** Extracts the exact filament length/weight and print duration.
5.  **Report:** Updates the Job Status in NovaMake with the estimates and a "Printability Score" (0-100%).

-----

## ✨ Key Features

### 1\. Geometric Health Check

Before a printer heats up, we verify the math.

  * **Manifold Check:** Are there holes in the mesh?
  * **Thin Walls:** Are parts of the model too thin for the nozzle diameter?
  * **Overhang Detection:** Does the model require support structures? (Adds to cost/waste).

### 2\. Multi-Engine Support

Different manufacturing technologies require different logic. The worker supports pluggable backends:

  * **FDM (Filament):** Uses *CuraEngine* logic.
  * **SLA (Resin):** Uses volumetric analysis.
  * **CNC:** (Future) Verifies toolpaths against stock material size.

### 3\. The "Eco-Quote" Calculator

This is critical for the Circular Economy.

  * It calculates the **Support Waste Ratio**: "This model is 100g, but requires 50g of support material."
  * It flags inefficient designs to the user: "Rotate model 90° to save 30% material."

-----

## 🚀 Getting Started

### Prerequisites

  * Docker Desktop
  * Python 3.11+
  * **System Deps:** `libgeos-dev` (for geometry calculations)

### Installation

1.  **Clone the repo:**
    ```bash
    git clone https://github.com/novaeco-tech/novamake-worker-slicer-check.git
    cd novamake-worker-slicer-check
    ```
2.  **Start the Dev Environment:**
    ```bash
    make dev
    ```
      * Starts the worker container with pre-installed slicing binaries.
      * Starts a local RabbitMQ.
      * **Health Check:** `http://localhost:8080/health`

### Configuration (`.env`)

```ini
# Queue
RABBITMQ_URI=amqp://guest:guest@rabbitmq:5672/
QUEUE_NAME=queue.make.slicer-jobs

# Object Storage (for fetching STLs)
S3_ENDPOINT=http://minio:9000
S3_BUCKET=make-models

# Slicer Settings
DEFAULT_LAYER_HEIGHT=0.2
TEMP_DIR=/tmp/slicer
```

-----

## 📂 Repository Structure

```text
novamake-worker-slicer-check/
├── src/
│   ├── main.py             # Consumer Loop
│   ├── engines/            # Wrappers for Cura/Slic3r binaries
│   ├── analysis/           # Trimesh/NumPy geometry logic
│   └── models/             # Printer Profile definitions (JSON)
├── tests/                  # Pytest suite
├── samples/                # Sample STL files for testing (Benchy, Cube)
└── Dockerfile              # Builds the container with C++ deps
```

-----

## 🧪 Testing

We use **Snapshot Testing** for G-code generation.

  * **Geometry Tests:** `make test-geo`
      * Loads `samples/broken_mesh.stl` and asserts that the worker returns `valid: false`.
  * **Slicing Tests:** `make test-slice`
      * Slices `samples/calibration_cube.stl` and compares the output G-code stats against a known baseline (e.g., "Must use 12.5g of filament").

-----

## 🤝 Contributing

We need contributors with experience in **Computational Geometry**, **C++**, and **3D Printing**.
See [CONTRIBUTING.md](https://www.google.com/search?q=../.github/CONTRIBUTING.md) for details.

**Maintainers:** `@novaeco-tech/maintainers-sector-novamake`