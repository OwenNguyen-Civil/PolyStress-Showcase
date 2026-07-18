# Polystress-Py: Advanced Structural Section Analysis

**Product development life cycle (PDLC):** 1/2026 – 4/2026
**Status:** Completed

![Python](https://img.shields.io/badge/Python-3.8%2B-blue)
![License](https://img.shields.io/badge/License-MIT-green)
![Status](https://img.shields.io/badge/Status-Completed-success)

**Polystress-Py** is a computational structural engineering tool designed to analyze arbitrary cross-sections imported directly from CAD (`.dxf`).

Unlike conventional rasterization or pixel-counting approaches, this tool utilizes **Green's Theorem** for geometric property calculation and a custom **scanline / ray-casting algorithm** for shear stress distribution. The method aims to eliminate rasterization-related geometric errors while maintaining a low memory footprint.

---

## 🚀 Key Features

* **CAD Interoperability:** Direct import of geometry (Polylines and Circles) from `.dxf` files using `ezdxf`. Polygon boundaries are classified using a signed-area convention: the largest polygon is treated as the outer shell, while remaining polygons are treated as internal holes.
* **Arbitrary Shape Analysis:** Calculates cross-sectional properties such as $A$, , $Cx$, $Cy$, $I_x$, $I_y$, and $I_{xy}$ for arbitrary polygonal geometries using **Green's Theorem** and boundary integrals.
* **Scanline-Based Shear Stress Analysis:** Uses horizontal geometric intersections to determine the effective cut width $b$ and first moment of area $Q$ for shear stress calculations.
* **Direct Stress Field Evaluation:** Evaluates normal stress, shear stress, and Von Mises stress over a dense set of points within the cross-section.
* **Stress Field Visualization:** Uses Delaunay triangulation and piecewise linear interpolation to visualize the resulting Von Mises stress field.
* **Low Memory Footprint:** Avoids storing a full pixel-based representation of the cross-section.

---

## 📐 Theoretical Background

Polystress-Py is based on the analysis of stress distributions over arbitrary two-dimensional cross-sections.

The general computational workflow consists of:

$$
\text{Geometry}
\rightarrow
\text{Section Properties}
\rightarrow
\text{Applied Loads}
\rightarrow
\text{Stress Evaluation}
\rightarrow
\text{Von Mises Stress}
$$

The section properties are calculated from the geometric boundary rather than by counting pixels or relying on a conventional finite element stiffness matrix.

The main quantities involved include:

* Cross-sectional area $A$
* Centroid coordinates
* Second moments of area $I_x$ and $I_y$
* Product of inertia $I_{xy}$
* Normal stress
* Shear stress
* Von Mises stress

For example, the normal stress field may be evaluated using a combination of axial force and bending contributions:

For a general biaxial bending case, the normal stress field is evaluated using the axial force and the coupled section properties:

$$
\sigma(x, y) = \frac{N}{A} + \frac{M_x I_y + M_y I_{xy}}{I_x I_y - I_{xy}^2}y + \frac{M_y I_x + M_x I_{xy}}{I_x I_y - I_{xy}^2}x
$$


The shear stress calculation is based on the classical relationship:

$$
\tau = \frac{VQ}{Ib}
$$

where the geometric quantities $Q$, $I$, and $b$ are determined from the cross-sectional geometry.

---

## 🧠 Algorithmic Logic

### 1. Signed Polygon Convention for Holes

The geometry preprocessing stage uses a signed-polygon convention to represent outer boundaries and internal holes.

The largest polygon is treated as the outer shell and normalized to counter-clockwise (CCW) orientation. Remaining polygons are treated as internal holes and normalized to clockwise (CW) orientation.

This produces the following convention:

- Outer shell: CCW orientation → positive signed area
- Internal hole: CW orientation → negative signed area

This orientation convention allows Green's Theorem to naturally account for holes through signed geometric contributions. As a result, net geometric properties can be calculated by summing the contributions of all boundaries:

$$
\text{Net Property} = \text{Shell Contribution} + \text{Hole Contributions}
$$

Because the hole boundaries have the opposite orientation, their contributions are naturally negative.

The current implementation assumes that the largest polygon represents the outer boundary and that all remaining polygons represent internal holes.

### 2. Geometric Properties — Green's Theorem

Instead of meshing the area, Polystress-Py integrates along the boundary path.

This allows the program to calculate geometric properties such as area, centroid, and moments of inertia directly from the polygonal boundary.

The approach is particularly effective for arbitrary polygonal cross-sections and avoids the geometric discretization associated with pixel-based methods.

---

### 3. The geometric intersections along each scanline are calculated directly from the polygon boundaries.

Early versions of the project utilized a pixel-counting method to determine the cut width $b$ for shear stress calculations.

At each evaluation height, the algorithm computes all intersections between the horizontal scanline and the section boundaries.

The intersection coordinates are then sorted along the x-axis. According to the even–odd parity rule, consecutive pairs of intersections define intervals inside the cross-section.

For example:

```text

x₀────────x₁      x₂──────x₃
│          │      │        │
    solid              solid

```
However, this introduced rasterization and aliasing errors near boundaries. For example, a thin-walled section could produce an incorrect local width because the pixel grid did not exactly represent the actual geometry.

The current approach uses a scanline / ray-casting procedure:

1. A horizontal scanline is evaluated at a given $y$-coordinate.
2. The algorithm calculates intersections between the scanline and polygon edges.
3. Intersection points are sorted along the $x$-axis.
4. Solid intervals are identified and regions corresponding to holes are subtracted.
5. The resulting geometric intervals are used to calculate section width and related quantities.

The intersection coordinates are obtained from the linear interpolation of polygon edges:

$$
x = x_1 + (y-y_1)\frac{x_2-x_1}{y_2-y_1}
$$

This approach eliminates the pixelization error associated with raster-based geometry detection.

### 4. Direct Numerical Integration of the First Moment of Area ($Q$)

For shear stress evaluation, the first moment of area $Q$ is calculated directly by integrating horizontal differential strips above the evaluation level.

At each sampled height $y$, the scanline algorithm determines the total solid width:

$$
b(y) = \sum \left(x_{\mathrm{right}} - x_{\mathrm{left}}\right)
$$

The corresponding differential area is then:

$$
dA = b(y)\.dy
$$

The first moment contribution of this strip about the centroidal $x$-axis is:

$$
dQ = (y - C_y)\.dA
$$

The algorithm accumulates these contributions from the top of the cross-section downward:

$$
Q(y) = \int_y^{y_{\max}} (y' - C_y)\.b(y')\.dy'
$$

In the numerical implementation, this integral is approximated using discrete horizontal slices:

$$
Q(y_i) \approx \sum_{k=i}^{n} (y_k - C_y)\.b(y_k)\.\Delta y
$$

This approach directly evaluates the first moment of area from the geometrically reconstructed cross-section width, without explicitly generating a finite element mesh or pixel-based representation.

---

## 🔬 Computational Workflow

The overall computational pipeline is:

```text
DXF Geometry
     ↓
Entity Extraction
     ↓
Geometry Cleaning
     ↓
Polygon Sorting by Absolute Area
     ↓
Signed Orientation Normalization
     ↓
Outer Shell / Hole Classification
     ↓
Green's Theorem
     ↓
Section Properties
     ↓
Scanline / Ray-Casting Evaluation
     ↓
Stress Field Calculation
     ↓
Dense Point Evaluation
     ↓
Delaunay Triangulation
     ↓
Piecewise Linear Interpolation
     ↓
Von Mises Stress Visualization
```

The triangulation stage is used exclusively to construct a continuous visual representation of the calculated stress field.

It does **not** constitute a finite element mesh and is not used to assemble or solve a stiffness matrix.

---

## 🎨 Stress Field Evaluation and Visualization

Polystress evaluates the analytical/numerical stress field on a dense set of points within the cross-section, then uses a **Delaunay triangulation** and **piecewise linear interpolation** to visualize the resulting Von Mises stress field.

The process can be summarized as:

```text
Analytical / Numerical Stress Field
              ↓
      Dense Evaluation Points
              ↓
      Delaunay Triangulation
              ↓
 Piecewise Linear Interpolation
              ↓
      Von Mises Visualization
```

The stress values are first calculated at the evaluation points. The triangulation and interpolation are subsequently used to construct a continuous visualization of the resulting field.

---

## ⚠️ Assumptions & Limitations

### Alternative Approach to FEM

Polystress-Py is **not intended to replace the Finite Element Method (FEM)**.

Instead, this project explores an alternative computational approach based on:

* Analytical geometry
* Boundary integration
* Numerical integration
* Direct stress-field evaluation
* Geometric intersection algorithms

The project is therefore better understood as a computational structural mechanics tool based on direct geometric and numerical methods rather than as a conventional finite element solver.

---

### Polygonal Geometry Representation

The current implementation is most reliable for polygonal geometries, including sections with polygonal holes such as square or rectangular openings.

Circular geometries and curved boundaries are currently approximated by polygonal representations.

For example, a circular boundary may be represented by a 72-sided polygon. Depending on the geometry and the evaluated quantity, this approximation can introduce significant errors. In certain cases, deviations may reach approximately 15%.

Therefore, circular holes and curved boundaries should currently be treated as an approximation rather than an exact geometric representation.

---

### Scanline / Ray-Casting Limitations

The scanline algorithm evaluates the cross-section by determining intersections between horizontal scanlines and the section boundaries.

For geometries containing curved boundaries or complex solid–void configurations, the intersection-based reconstruction may encounter difficulties in correctly identifying or pairing all boundary intersections.

This can occur particularly near complex geometric transition zones, where a scanline crosses between solid and void regions.

Such cases may introduce local errors in:

- The reconstructed section width $b(y)$
- The first moment of area $Q(y)$
- The resulting shear stress distribution

The accuracy of the shear stress field depends directly on the accuracy of the reconstructed width function $b(y)$. Any missed or incorrectly paired scanline intersections can produce errors in $b(y)$, which propagate into the numerical integration of $Q(y)$ and consequently affect the calculated shear stress.

In addition, the numerical evaluation of $Q(y)$ is performed using discrete horizontal slices with a finite spacing $\Delta y$. This introduces a discretization error that depends on the selected vertical resolution.

Improving the robustness of solid–void transition detection and reducing discretization error through adaptive resolution are potential areas for future development.

---

### Stress Field Interpolation

The calculated stress field is evaluated at a dense set of points and then visualized using Delaunay triangulation and piecewise linear interpolation.

The resulting visualization is therefore dependent on:

* The density of evaluation points
* The spatial distribution of those points
* The triangulation topology
* The interpolation between calculated values

The interpolation process is used for visualization and does not generate additional physical information beyond the evaluated stress field

---

### Current Scope

The current project is primarily intended for:

* Two-dimensional cross-sectional analysis
* Arbitrary polygonal sections
* Sections containing polygonal holes
* Exploratory computational structural mechanics
* Visualization of stress distributions

The current implementation does not attempt to model:

* Three-dimensional structural behavior
* Material nonlinearity
* Plasticity
* Fracture
* General nonlinear mechanics
* Full finite element analysis

---

## 📊 Verification & Validation

The numerical results of Polystress-Py should be verified against analytical solutions or independent reference calculations.

Suitable benchmark cases include:

### Rectangular Cross-Section

For a rectangular section:

$$
A = bh
$$

$$
I_x = \frac{bh^3}{12}
$$

$$
I_y = \frac{hb^3}{12}
$$

The calculated results can be compared with these analytical values.

---

### Circular Cross-Section

For an ideal circular section:

$$
A = \pi r^2
$$

$$
I_x = I_y = \frac{\pi r^4}{4}
$$

Because the current implementation approximates curved boundaries using polygonal representations, this case is also useful for evaluating the error introduced by geometric discretization.

---

### Relative Error

The relative error is calculated as:

**Relative Error (%) = |Polystress Result - Reference Result| / |Reference Result| × 100**

## 🛠️ Installation

Clone the repository:

```bash
git clone https://github.com/OwenNguyen-Civil/PolyStress.git
cd PolyStress
```

Install the required dependencies:

```bash
pip install -r requirements.txt
```

Main dependencies include:

* `numpy`
* `matplotlib`
* `ezdxf`

---

## 💻 Usage

### 1. Prepare the CAD Geometry

Create the cross-section in AutoCAD or another compatible CAD program.

Current recommendations:

* Use units: **mm**
* Place the section on the designated layer, currently `'mod'`
* Ensure the geometry consists of closed `LWPOLYLINE` or `CIRCLE` entities
* Save the geometry as a compatible DXF file

---

### 2. Run the Analysis

If the project is implemented as a Python script:

```bash
python polystress.py
```

If the analysis is currently distributed as a Jupyter Notebook, open and run:

```text
Polystress.ipynb
```

---

### 3. View the Results

The program outputs section properties and stress-related results to the console and generates visualizations of:

* Normal stress
* Shear stress
* Von Mises stress

---

## 🔄 Evolution from Legacy Code

This project is the modern evolution of my original research in **MATLAB**.

### Old Version

The original implementation was developed in MATLAB:

**MATLAB-STRUCTURAL-ANALYSIS**

### Improvements in the Python Version

The Python version:

* Removes the dependency on a proprietary MATLAB license
* Introduces direct CAD geometry integration
* Uses `ezdxf` for DXF processing
* Implements an optimized scanline / ray-casting approach
* Provides automated stress-field visualization
* Uses a lower-memory geometry evaluation strategy

---

## 📂 Project Structure

```text
Polystress-Py/
├── core
    └──solver.py       # Main calculation engine
├── geometry
    └──dxf_parser.py   # Processing geometry from the .dxf         
├── postprocess
    ├──report.py       #Showing calculation result
    └──visualization.py  #Visualizing         
├──LICENSE
└── README.md                  # Project documentation
```

---

## 🚧 Future Development

Potential areas for future development include:

* [ ] Improve curved boundary representation
* [ ] Support exact circular geometry
* [ ] Improve solid–void transition detection
* [ ] Develop adaptive scanline evaluation
* [ ] Add more automated analytical validation cases
* [ ] Support additional DXF entities
* [ ] Improve handling of complex geometries
* [ ] Improve stress-field sampling and interpolation
* [ ] Develop a more structured Python package interface

---

## 📜 License

This project is licensed under the MIT License.

---

**Author:** Owen Nguyen / Nguyen Phuoc Thinh
*Aspiring Computational Structural Engineer*

---

This is my first project — and in many ways, my first “child.”

I hope it can help you get through Strength of Materials, or serve as a case study for anyone starting out in FEA or computational engineering, just like I once did.
