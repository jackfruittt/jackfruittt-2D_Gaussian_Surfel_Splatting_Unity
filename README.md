# 2D Gaussian Surfel Splatting (Unity)

An approach to drawing a live RGB-D point cloud as little Gaussian disks (surfels) in Unity
instead of dots. Each depth pixel gets a small disk that lies on the surface, which fills the
gaps between points so it looks solid instead of like a cloud of dots.

This repo is a writeup of the approach and results. The code isn't published.

### Why

Point clouds go sparse when you get close or look from the side. The dots spread out and you
see through them. Disks instead of dots cover the gaps, so the surface holds up from those
off-axis views (close up, side-on) where a normal streamed cloud just falls apart.

### What this is

Surfel splatting, the old-school kind. Each splat is a flat 2D Gaussian sitting on the surface,
made straight from the depth and colour frames.

Not the trained 2DGS / 3DGS people use for novel-view synthesis. Nothing is trained or
optimized, no network, no spherical harmonics. Position, size, orientation and colour all come
off the sensor frames. "2D" just means a flat disk, not a 3D blob.

## Pipeline

Three stages, all on the GPU (no readback to the CPU per frame).

**1. Ingest.** Upload the depth and colour frames into GPU textures, pass the camera intrinsics.

**2. Depth to surfels (compute shader).**
- **Bilateral filter:** denoise depth, keep the real edges.
- **Kalman filter:** per-pixel smoothing over time. Steady when still, resets on motion so
  things don't smear.
- **Back-projection:** depth to a 3D position.
- **Normals:** finite differences off the neighbouring points.
- **Edge detection:** depth jumps face the camera instead of stretching across the gap.
- **Tangent + anisotropy:** how much to stretch/tilt each disk so it matches the surface.

**3. Rasterize (vertex + fragment shader).** Each point becomes a quad, turned to face the
surface and stretched into an ellipse, with a Gaussian falloff so the edges are soft. Drawn
opaque and let the depth buffer sort out overlaps, so no per-frame sorting and no transparency
mess. MSAA cleans up the edges.

Two modes: the oriented disks, or plain camera-facing quads if you want to compare.

## Results

Media coming.
