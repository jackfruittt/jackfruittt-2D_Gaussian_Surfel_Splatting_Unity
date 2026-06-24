# Streaming RGB-D Splat Primitives in Unity

A comparison of rendering primitives for a live RGB-D point cloud in Unity: plain points, flat
quads, camera-facing billboards, and surface-oriented 2D Gaussian surfels. Each depth pixel is
turned into a primitive every frame, straight from the sensor data, and the primitives are
compared on visual completeness and render cost.

This repository documents the approach and results; the source is not published.

### Motivation

This started as a 2D Gaussian surfel renderer. Streamed point clouds go sparse at close range
and at oblique angles (the points separate and the surface reads as gaps), and orienting a small
disk to the local surface looked like the way to keep it continuous. As more of the literature
came into view, the question turned comparative rather than advocative: how much does the choice
of primitive actually matter for live RGB-D, and is the surface-oriented surfel worth its cost
over simpler options? That comparison is what this repo documents.

### Scope

Every primitive is generated directly from the depth and colour frames, per frame, with no
training, no network, no spherical harmonics, and no per-scene optimisation. This is the live,
sensor-driven regime, not the trained novel-view-synthesis regime that 3DGS / 2DGS target. "2D
Gaussian" here means the planar disk primitive (a flat 2D Gaussian on the surface), as opposed
to a volumetric 3D Gaussian.

### Background

The link between billboards and 2D Gaussian splats is already recognised in the literature.
Weiss & Bradley's *Gaussian Billboards* (DisneyResearch, 2024) and Svitov et al.'s *BBSplat*
both note that a 2DGS primitive is essentially a billboard, a flat oriented scaled 2D quad, with
a Gaussian-modulated opacity, and extend 2DGS with per-splat textures. Those are trained,
offline reconstruction methods aimed at novel-view-synthesis quality. This project sits in the
opposite corner: no training, real-time, fed by a live depth sensor. It borrows the primitives,
not the optimisation, and asks which one is actually worth using in that streaming setting.

## Pipeline

Three stages, all GPU-side (no per-frame CPU readback).

**1. Ingest.** Depth and colour frames are uploaded to GPU textures; camera intrinsics are
passed in.

**2. Depth to surfels (compute shader).**
- **Kalman filter:** per-pixel temporal smoothing of depth; stable when static, reset on motion
  to avoid smearing.
- **Back-projection:** depth to a 3D position.
- **Normals:** finite differences over neighbouring points.
- **Edge detection:** depth discontinuities are given a fixed forward-facing normal (flattened
  to a billboard) rather than tilted across the gap.
- **Tangent + anisotropy:** per-axis disk stretch and orientation from the local surface.

**3. Rasterise (vertex + fragment shader).** Each point is expanded to a quad, oriented to the
surface and stretched into an ellipse, with a Gaussian falloff. Rendered opaque with overlaps
resolved by the depth buffer (no per-frame sorting, no transparency ordering), with MSAA on
edges.

Two modes are provided: the surface-oriented disks above, and a simpler baseline that draws each
point as a flat quad, either camera-facing (a billboard, always facing the viewer) or offset in
the sensor image plane (which foreshortens edge-on).

## Complexity

N = primitives per frame (depth pixels after decimation). Everything in this repo is O(N) per
frame with no sort and no training; the primitives differ only by constant factors. The trained
methods are the opposite regime: a per-scene optimisation, plus a per-frame depth sort for the
alpha-blended ones.

| approach | per-frame | orient | fragment | sort | train |
|---|---|---|---|---|---|
| point / flat quad | O(N) | none | flat colour | none | none |
| camera-facing billboard | O(N) | view-facing | flat colour | none | none |
| 2D Gaussian surfel | O(N) | surface (O(1)/px) | Gaussian | none | none |

## Results

A pre-planned route was recorded and replayed for each stream to keep the viewing path
consistent between runs. All settings were held constant; only the feature under test was
enabled/disabled. Object detection was CPU-bound throughout (no GPU/OpenVINO), so that overhead
is included in its figures.

Three primitives are compared: a flat image-plane quad (the naive baseline), surface-oriented
2D Gaussian surfels, and a camera-facing billboard.

The flat quad foreshortens at oblique angles and leaves visible gaps up close and side-on. Both
the surfels and the billboard remove this. The surfels run at roughly 30 to 45 render fps; the
billboard is comparable (about 30 to 40) and holds up visually while being far simpler. The flat
quad is faster (40 to 60) only by doing and showing less.

In short, for raw coverage the two are close, but the surface orientation does show up as better
object shape: 2DGS holds form and occludes correctly as the view moves off-axis, where the
billboard faces the viewer and reads more like a flat card. That shape edge costs extra compute
and adds some temporal instability. The billboard figures are also unoptimised (it still renders
in a transparent, depth-write-off pass), so it could be made faster still.

The camera stream rate is unchanged across modes, as expected (sensor-bound, not render-bound).
All clips loop. The 2D Gaussian (Kalman) and billboard clips play at 1x; the other clips are
1.5x. The billboard was captured with the Kalman filter only.

### Billboard vs 2D Gaussian (Kalman)

<div align="center">
<table width="100%">
<tr>
<td width="50%" align="center"><b>Camera-facing billboard</b><br><img src="media/plc_billboard_kf_enabled.webp" alt="Camera-facing billboard, Kalman" width="100%"></td>
<td width="50%" align="center"><b>2D Gaussian surfels</b><br><img src="media/2dgs_kf_enabled.webp" alt="2D Gaussian surfels, Kalman" width="100%"></td>
</tr>
</table>
</div>

### Flat image-plane quad (naive baseline)

**No Kalman**
<p align="center"><img src="media/plc_no_kf.webp" alt="Flat quad, no Kalman" width="600"></p>

**Kalman**
<p align="center"><img src="media/pcl_kf_enabled.webp" alt="Base point cloud, Kalman" width="600"></p>

**Kalman + object detection**
<p align="center"><img src="media/pcl_kf_objd_enabled.webp" alt="Base point cloud, Kalman + object detection" width="600"></p>

### 2D Gaussian surfel splatting

**No Kalman**
<p align="center"><img src="media/2dgs_no_kf.webp" alt="2D Gaussian surfels, no Kalman" width="600"></p>

**Kalman**
<p align="center"><img src="media/2dgs_kf_enabled.webp" alt="2D Gaussian surfels, Kalman" width="600"></p>

**Kalman + object detection**
<p align="center"><img src="media/2dgs_kf_objd_enabled.webp" alt="2D Gaussian surfels, Kalman + object detection" width="600"></p>

### Camera-facing billboard

**Kalman**
<p align="center"><img src="media/plc_billboard_kf_enabled.webp" alt="Camera-facing billboard, Kalman" width="600"></p>

### Settings
<p align="center"><img src="media/settings.png" alt="Inspector settings" width="600"></p>

## Conclusion

None of this is new. The primitives are all known, and 2DGS being basically a billboard with a
Gaussian falloff is already in the literature. The work was building them in Unity and comparing
them like-for-like on a live RGB-D stream, no training, no fitted scene.

It depends. Both clearly beat the flat quad. For just filling the surface, a camera-facing
billboard does about as well as 2DGS, it's simpler, and it doesn't flicker like the tilted disks.
It's slightly slower right now, but only because it's drawn transparent, so nothing gets z-culled
and every overlapping fragment gets shaded. The billboard itself is cheaper, drawing it opaque
would make it faster than 2DGS.

2DGS still wins on one thing: shape. The disks sit on the actual surface, so objects keep their
shape and block each other correctly as you move around them. Billboards always face the camera,
so off-angle they look flatter, more like cards. So orienting the splats does buy something,
better 3D shape, just at the cost of more compute and some instability.

Rough rule of thumb:
- **Billboard** if you mainly want a complete, stable, cheap point cloud viewed roughly head-on,
  on tight hardware (e.g. Quest).
- **2DGS** if you're orbiting objects and care about their shape and occlusion from off-axis, and
  can spend the extra compute.

All built from scratch in Unity, shaders not published here.

## References

- Pfister et al., *Surfels: Surface Elements as Rendering Primitives*, SIGGRAPH 2000.
- Zwicker et al., *Surface Splatting*, SIGGRAPH 2001; *EWA Splatting*, IEEE TVCG 2002.
- Kerbl et al., *3D Gaussian Splatting for Real-Time Radiance Field Rendering*, SIGGRAPH 2023. The trained radiance-field method this is positioned against.
- Huang et al., *2D Gaussian Splatting for Geometrically Accurate Radiance Fields*, SIGGRAPH 2024. The oriented-disk primitive.
- Weiss & Bradley, *Gaussian Billboards: Expressive 2D Gaussian Splatting with Textures*, DisneyResearch, arXiv 2024. Draws the 2DGS-equals-billboard link; adds per-splat textures (trained, offline).
- Svitov et al., *BBSplat: Learnable Textured Primitives for Efficient 2D Gaussian Splatting*, arXiv 2024. <https://github.com/david-svitov/BBSplat>
