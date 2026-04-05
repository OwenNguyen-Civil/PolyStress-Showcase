# Polystress-Py: Advanced Structural Section Analysis
Product development life cycle (PDLC): 1/2026 - 4/2026 /
Status: Completed

![Python](https://img.shields.io/badge/Python-3.8%2B-blue)
![License](https://img.shields.io/badge/License-MIT-green)
![Status](https://img.shields.io/badge/Status-Active-success)

**Polystress-Py** is a computational structural engineering tool designed to analyze arbitrary cross-sections imported directly from CAD (.dxf). 

Unlike standard "pixel-counting" methods, this tool utilizes **Green's Theorem** for geometric properties and a custom **Exact Geometric Slicing (Ray Casting)** algorithm for shear stress distribution, ensuring absolute precision ($b$ error = 0) while maintaining extremely low memory footprint.


## 🚀 Key Features

* **CAD Interoperability:** Direct import of geometry (Polylines & Circles) from `.dxf` files using `ezdxf`. Automatically detects "Outer Shell" vs "Inner Holes".
* **Arbitrary Shape Solver:** Calculates properties ($Area, I_x, I_y, I_{xy}$) for any closed shape using **Green's Theorem** (Boundary Integral).
* **Exact Shear Flow Analysis:** * Replaces traditional grid-based pixelization (which is memory-heavy and prone to aliasing errors at edges).
    * Uses a **Scanline/Ray Casting algorithm** to calculate the exact cut width ($b$) and First Moment of Area ($Q$) at any height.
    * Reduces RAM usage by ~90% compared to mesh-based approaches.
* **Visualization:** Generates high-resolution stress heatmaps (Normal, Shear, Von Mises) using `matplotlib`.

## 🧠 Algorithmic Logic

### 1. Geometric Properties (Green's Theorem)
Instead of meshing the area, the tool integrates along the boundary path. This allows for extremely fast calculation of moments of inertia, regardless of the section's complexity.

### 2. The "Exact Slicing" Optimization
Early versions utilized a pixel-counting method to determine the cut width $b$ for shear stress calculations ($\tau = \frac{VQ}{Ib}$). However, this led to "aliasing" errors where the width was overestimated at the edges (e.g., measuring 56mm instead of 42mm for thin-walled sections).

**The Solution:** I implemented a **Ray Casting algorithm**:
1.  At each $y$ integration step, a virtual ray is cast horizontally.
2.  The algorithm solves for exact intersection points ($x = x_1 + (y-y_1)\frac{x_2-x_1}{y_2-y_1}$) with all polygon edges.
3.  Segments inside "holes" are logically subtracted.
4.  **Result:** Zero-error geometry detection and significantly faster performance.

## 🛠️ Installation

1.  Clone the repository:
    ```bash
    git clone (https://github.com/OwenNguyen-Civil/PolyStress)
    cd Polystress.ipynb
    ```

2.  Install dependencies:
    ```bash
    pip install -r requirements.txt
    ```
    *(Dependencies: `numpy`, `matplotlib`, `ezdxf`)*

## 💻 Usage

1.  **Prepare CAD:** Draw your section in AutoCAD/Revit.
    * Use units: **mm**.
    * Place the section on a specific layer (default: `'mod'`).
    * Ensure shapes are closed `LWPOLYLINE` or `CIRCLE`.
    * Save as **DXF 2010** (or later).
2.  **Run Analysis:**
    ```bash
    python Polystress.ipynb
    ```
3.  **View Results:** The script will output detailed properties to the console and display a stress contour plot.

## 🔄 Evolution from Legacy Code

This project is the modern evolution of my original research in **Matlab**. 
* **Old Version:** [(https://github.com/OwenNguyen-Civil/MATLAB-STRUCTURAL-ANALYSIS)]
* **Improvements:** This Python version removes proprietary dependencies (Matlab license), introduces direct CAD integration, and features the optimized Slicing Algorithm.

## 📂 Project Structure
Polystress-Py/ ├── polystress.py # Main calculation engine ├── Section.dxf # Example CAD file ├── requirements.txt # Python dependencies └── README.md # Documentation


---
**Author:** [Owen Nguyen]  
*Aspiring Computational Structural Engineer*
