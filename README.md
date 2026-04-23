# Homework 4 Writeup

## Overview

In this project, I built both a cloth simulation system and a rendering pipeline. The cloth was modeled as a grid of point masses connected by structural, shearing, and bending springs. Its motion was simulated using external forces, Hooke’s law spring forces, damped Verlet integration, and spring length constraints. I then added collisions with spheres and planes, followed by self-collision using spatial hashing so the cloth could fold on itself without clipping. Finally, I implemented several GLSL shaders to render the cloth with diffuse shading, Blinn-Phong shading, texturing, bump mapping, displacement mapping, and mirror reflection.

What I found especially interesting in this assignment was the connection between physics and rendering in computer graphics. Small variations in simulation parameters such as stiffness, density, and damping had a large effect on realism, while the choice of shader completely changed the visual appearance of the cloth. This assignment also made the importance of efficiency very clear, especially in the self-collision portion where spatial hashing made the problem practical.

---

## Part 1

### Pinned Cloth Wireframe
![Pinned cloth wireframe](images/part1_pinned4_wireframe.png)

### Constraint Comparison

<table>
  <tr>
    <td align="center">
      <img src="images/part1_no_shearing.png" alt="Without shearing constraints" width="300"><br>
      <b>Without shearing constraints</b>
    </td>
    <td align="center">
      <img src="images/part1_only_shearing.png" alt="Only shearing constraints" width="300"><br>
      <b>Only shearing constraints</b>
    </td>
    <td align="center">
      <img src="images/part1_all_constraints.png" alt="All constraints enabled" width="300"><br>
      <b>All constraints enabled</b>
    </td>
  </tr>
</table>

---

## Part 2

For this part, I ran a series of experiments in the cloth simulator by pausing the simulation with `P`, changing one parameter at a time in the GUI, and then resuming. The simulator computes the motion of the cloth by first calculating the mass per point mass as

\[
\frac{\text{width} \cdot \text{height} \cdot \text{density}}{\text{num\_width\_points} \cdot \text{num\_height\_points}}
\]

It then sums the external acceleration forces to form the external force \(F = ma\), applies Hooke’s law spring forces, and updates positions using damped Verlet integration. In my implementation, the bending spring force is weaker than the others because I multiply it by `0.2 * ks`. The damping term appears in the position update as

\[
(1 - \text{damping}/100)(x_t - x_{t-\Delta t})
\]

### Effect of `ks` (Spring Constant)

Changing `ks` has the strongest influence on how stiff the cloth feels because the spring force is directly proportional to

\[
ks \cdot (\text{current length} - \text{rest length})
\]

A larger `ks` makes the springs resist deformation more strongly, while a smaller `ks` makes the cloth more compliant. When `ks` is very low, the cloth looks loose and stretchy: it sags more under gravity, stretches more between pinned points, and settles into a softer resting shape. When `ks` is very high, the cloth behaves more like a stiff fabric: it resists stretching, stays closer to its initial shape as it falls, and reaches a resting state with less sag.

<table>
  <tr>
    <td align="center">
      <img src="images/part2_low_ks.png" alt="Low spring constant" width="360"><br>
      <b>Low ks</b>
    </td>
    <td align="center">
      <img src="images/part2_high_ks.png" alt="High spring constant" width="360"><br>
      <b>High ks</b>
    </td>
  </tr>
</table>

### Effect of Density

Density changes the amount of mass assigned to each point mass, so a higher density makes the cloth feel heavier. Although gravity is still applied uniformly, the spring-force-to-mass ratio changes, so high-density cloth responds more sluggishly than low-density cloth. In practice, increasing density made the cloth fall more heavily, sag farther below the pinned points, and settle into a lower final shape. Lower density produced a lighter cloth that deformed less and came to rest in a higher position.

<table>
  <tr>
    <td align="center">
      <img src="images/part2_low_density.png" alt="Low density" width="360"><br>
      <b>Low density</b>
    </td>
    <td align="center">
      <img src="images/part2_high_density.png" alt="High density" width="360"><br>
      <b>High density</b>
    </td>
  </tr>
</table>

### Effect of Damping

Changing damping affects the inertial term in the Verlet update through the multiplier `(1 - damping / 100)`. With low damping, the contribution from the previous time step is stronger, so the cloth oscillates more, swings longer, and takes more time to settle. With high damping, that contribution is reduced, so the cloth comes to rest faster and with less visible oscillation. I observed that low damping caused the cloth to overshoot equilibrium and wobble for longer, while high damping made it look much more overdamped. The final resting shape was usually similar when `ks` and density were held constant.

<table>
  <tr>
    <td align="center">
      <img src="images/part2_low_damping.png" alt="Low damping" width="360"><br>
      <b>Low damping</b>
    </td>
    <td align="center">
      <img src="images/part2_high_damping.png" alt="High damping" width="360"><br>
      <b>High damping</b>
    </td>
  </tr>
</table>

### Final Resting State

![Final shaded cloth resting state](images/part2_final_resting_state.png)

---

## Part 3

In Part 3, I implemented collisions with both spheres and planes by iterating through each point mass and checking it against every collision object at each time step. After the Verlet integrator updates the cloth positions, the simulator calls `collide()` on each collision primitive and adjusts the point mass position if it intersects the object.

For sphere collisions, I checked whether a point mass was inside or on the sphere by computing the distance from the point mass to the sphere center. If the distance was less than or equal to the sphere radius, a collision occurred. I then computed the tangent point by taking the direction from the sphere center to the point mass and extending it to exactly one radius. That point represents where the point mass should lie on the sphere surface. The correction vector was then computed from `last_position` to the tangent point, and the final position was updated using

\[
\text{last\_position} + (1 - \text{friction}) \cdot \text{correction}
\]

For plane collisions, I treated a collision as a point mass crossing from one side of the plane to the other during the last time step. I computed signed distances from both the current position and the previous position to the plane using the plane normal. If the signs differed, then the point mass had crossed the plane. I then found the line-plane intersection point, added a small `SURFACE_OFFSET` in the direction of the original side of the plane, and used that adjusted tangent point to compute the correction vector. As with the sphere, the new position became

\[
\text{last\_position} + (1 - \text{friction}) \cdot \text{correction}
\]

In `scene/sphere.json`, varying `ks` had a clear effect on the draping behavior. With the default `ks = 5000`, the cloth draped naturally around the sphere while still retaining stiffness. With `ks = 500`, the cloth became much softer, sagged farther down the sides, and formed deeper folds. With `ks = 50000`, the cloth behaved much more rigidly and tended to bridge over the sphere instead of wrapping closely around it.

<table>
  <tr>
    <td align="center">
      <img src="images/part3_sphere_ks500.png" alt="Sphere with ks 500" width="240"><br>
      <b>ks = 500</b>
    </td>
    <td align="center">
      <img src="images/part3_sphere_ks5000.png" alt="Sphere with ks 5000" width="240"><br>
      <b>ks = 5000</b>
    </td>
    <td align="center">
      <img src="images/part3_sphere_ks50000.png" alt="Sphere with ks 50000" width="240"><br>
      <b>ks = 50000</b>
    </td>
  </tr>
</table>

### Cloth Resting on the Plane

![Cloth resting on plane](images/part3_plane_resting.png)

---

## Part 4

In order to perform self-collision in Part 4, I used spatial hashing so that each point mass only checks collisions against nearby point masses instead of against all point masses in the cloth. A naive \(O(n^2)\) approach would become too expensive as the cloth resolution increases, so I partitioned space into 3D cells and stored, for each cell, the list of point masses inside it.

The implementation has three main parts. First, in `hash_position`, I computed the box dimensions \(w\), \(h\), and \(t\) from the cloth dimensions, then mapped each point mass position into a spatial cell using `floor(pos.x / w)`, `floor(pos.y / h)`, and `floor(pos.z / t)`. Combining these indices gave a unique hash key for that cell.

Second, in `build_spatial_map`, I cleared the previous map and inserted all point masses into the vector associated with their hash key.

Third, in `self_collide`, I looked up candidate neighbors in the same spatial cell, ignored the point mass itself, and checked whether any candidate was within `2 * thickness`. If so, I computed a correction vector that would separate the two point masses to exactly `2 * thickness`. I averaged all such corrections and scaled the result by `1 / simulation_steps` to improve stability and avoid abrupt jumps. During simulation, I called `build_spatial_map()` once per timestep and then called `self_collide(pm, simulation_steps)` for every point mass.

### Self-Collision Over Time

<table>
  <tr>
    <td align="center">
      <img src="images/part4_self_collision_stage1.png" alt="Initial self collision" width="240"><br>
      <b>Initial self-collision</b>
    </td>
    <td align="center">
      <img src="images/part4_self_collision_stage2.png" alt="Mid-stage folding" width="240"><br>
      <b>Mid-stage folding</b>
    </td>
    <td align="center">
      <img src="images/part4_self_collision_stage3.png" alt="Restful folded state" width="240"><br>
      <b>More restful state</b>
    </td>
  </tr>
</table>

### Effect of Density During Self-Collision

<table>
  <tr>
    <td align="center">
      <img src="images/part4_self_collision_low_density.png" alt="Low density self collision" width="360"><br>
      <b>Low density</b>
    </td>
    <td align="center">
      <img src="images/part4_self_collision_high_density.png" alt="High density self collision" width="360"><br>
      <b>High density</b>
    </td>
  </tr>
</table>

With density, increasing it made the cloth behave as if it were heavier. The cloth fell more aggressively, collapsed more quickly, and created denser pile-ups when interacting with the ground. Decreasing density made the cloth behave more lightly, so it fell more gently and formed a looser pile-up.

### Effect of `ks` During Self-Collision

<table>
  <tr>
    <td align="center">
      <img src="images/part4_self_collision_low_ks.png" alt="Low ks self collision" width="360"><br>
      <b>Low ks</b>
    </td>
    <td align="center">
      <img src="images/part4_self_collision_high_ks.png" alt="High ks self collision" width="360"><br>
      <b>High ks</b>
    </td>
  </tr>
</table>

With `ks`, decreasing it made the cloth softer and more flexible, so it bent more easily and conformed closely to its own folds. Increasing `ks` produced a stiffer material that resisted bending and bridging across itself, resulting in broader folds and a less compact pile-up.

---

## Part 5

A shader program is a small GPU program that computes how an object appears during rendering. In this project, there are two main shader stages: the vertex shader and the fragment shader. The vertex shader processes one vertex at a time, transforming geometric data such as positions, normals, tangents, and UV coordinates from model space to world space and passing values like `v_position`, `v_normal`, and `v_uv` to the next stage. The fragment shader then operates on each rasterized fragment, interpolates the per-vertex data, and computes the final color. This is where lighting, texturing, reflection, and other material effects are implemented.

The Blinn-Phong shading model approximates how light interacts with a surface by adding three components: ambient, diffuse, and specular. Ambient light represents general background illumination and prevents surfaces from appearing completely black. Diffuse light models matte reflection and depends on the cosine of the angle between the surface normal and the light direction. Specular light models shiny highlights and is based on the angle between the surface normal and the halfway vector between the light direction and the camera direction. In my shader, I combined ambient, diffuse, and specular terms to create the final Blinn-Phong appearance.

### Blinn-Phong Components

<table>
  <tr>
    <td align="center">
      <img src="images/part5_phong_ambient.png" alt="Ambient component" width="240"><br>
      <b>Ambient only</b>
    </td>
    <td align="center">
      <img src="images/part5_phong_diffuse.png" alt="Diffuse component" width="240"><br>
      <b>Diffuse only</b>
    </td>
    <td align="center">
      <img src="images/part5_phong_specular.png" alt="Specular component" width="240"><br>
      <b>Specular only</b>
    </td>
    <td align="center">
      <img src="images/part5_phong_full.png" alt="Full Blinn-Phong model" width="240"><br>
      <b>Full Blinn-Phong</b>
    </td>
  </tr>
</table>

### Custom Texture

![Custom texture render](images/part5_custom_texture.png)

### Bump vs. Displacement Mapping

To compare bump mapping and displacement mapping, I used the same non-default height texture in both cases. With bump mapping, the cloth and sphere appeared more detailed because the surface normals were modified in the shader, but the actual geometry did not change. With displacement mapping, the height texture altered the sphere geometry directly, moving the vertices according to the texture values and changing the silhouette. When I changed the sphere mesh resolution from coarse (`-o 16 -a 16`) to fine (`-o 128 -a 128`), bump mapping looked nearly the same at both resolutions because it only changes shading. Displacement mapping, however, improved significantly with the finer mesh because more vertices were available to displace.

<table>
  <tr>
    <td align="center">
      <img src="images/part5_displacement_res16.png" alt="Displacement mapping resolution 16" width="360"><br>
      <b>Displacement, resolution 16</b>
    </td>
    <td align="center">
      <img src="images/part5_displacement_res128.png" alt="Displacement mapping resolution 128" width="360"><br>
      <b>Displacement, resolution 128</b>
    </td>
  </tr>
</table>

<table>
  <tr>
    <td align="center">
      <img src="images/part5_bump_res16.png" alt="Bump mapping resolution 16" width="360"><br>
      <b>Bump, resolution 16</b>
    </td>
    <td align="center">
      <img src="images/part5_bump_res128.png" alt="Bump mapping resolution 128" width="360"><br>
      <b>Bump, resolution 128</b>
    </td>
  </tr>
</table>

### Bump Mapping on Cloth and Sphere

<table>
  <tr>
    <td align="center">
      <img src="images/part5_bump_cloth.png" alt="Bump mapping on cloth" width="360"><br>
      <b>Bump mapping on cloth</b>
    </td>
    <td align="center">
      <img src="images/part5_bump_sphere.png" alt="Bump mapping on sphere" width="360"><br>
      <b>Bump mapping on sphere</b>
    </td>
  </tr>
</table>

### Displacement Mapping

![Displacement mapping render](images/part5_displacement_normal.png)

### Mirror Shading

![Mirror shading render](images/part5_mirror_shading.png)
