---
icon: material/grid
---

# GOL Painter

**An algorithmic solver for inverse cellular automata states.**

[:material-web: Visit Website](https://gol-painter.com){: .md-button .md-button--primary }

---

!!! abstract "Project Overview"
    [GOL Painter](https://gol-painter.com) is an interactive, browser-based web application built around Conway's Game of Life. Unlike standard simulators that simply calculate future generations, this tool functions as an **inverse solver**. It accepts a user-drawn target image and programmatically computes the precise initial states required to naturally converge into that exact image after 1 or 2 computational steps.

## Core Mechanics & Impact

Computing the future state of a cellular grid is trivial. However, calculating the *preceding* state (the "Grandfather Problem") is a highly complex, non-reversible constraint satisfaction problem. 

I engineered GOL Painter to solve this mathematical challenge in real-time within a browser environment. For end-users, it acts as a fascinating interactive canvas. For engineering teams, it serves as a live demonstration of translating computationally heavy algorithmic state-space searches into a highly responsive, visual frontend.

---

## Technical Architecture

=== "Inverse Solving Algorithm"

    Finding a predecessor state in Conway's Game of Life requires searching through a massive matrix of possibilities. The engine utilizes algorithmic backtracking and constraint propagation to discard invalid predecessor states rapidly. This allows the system to resolve the necessary pixel configurations for 1-step and 2-step convergences without freezing the browser's main execution thread.

=== "State Management"

    The grid is not managed as a standard array of UI elements, which would incur massive rendering overhead. Instead, the cellular automaton state is managed through flattened, contiguous memory arrays. This ensures memory locality and allows the solving algorithm to iterate over bitwise states with maximum CPU efficiency.

=== "Real-time Rendering"

    Once the mathematical predecessor is calculated, the matrix data is dispatched to the rendering pipeline. The UI is decoupled from the algorithmic engine, ensuring that user interactions (drawing, scaling, and stepping) remain fluid even while the solver is computing complex multi-step convergences in the background.

---

## The Predecessor Problem

!!! info "Computational Complexity"
    In cellular automata, multiple different states can evolve into the exact same future state, but many states have *no* valid predecessor (known as "Garden of Eden" patterns). 


## Why it Matters

This project demonstrates my ability to tackle abstract, mathematically dense algorithmic problems and package them into intuitive, user-facing applications. It proves a core systems programming philosophy: **heavy computation should be invisible to the user.**