# Project 05: Python 3D Room Designer with Blender API

## Overview
Build a programmatic 3D room designer using Python and Blender's Python API (bpy) to generate, customize, and render photorealistic room designs. This project enables automated room generation, batch rendering of design variations, and professional-quality output for portfolio or client presentations.

## Learning Objectives
- Master Blender Python API (bpy) for 3D modeling
- Automate 3D scene creation and manipulation
- Implement procedural material systems
- Build rendering pipelines with Cycles/EEVEE
- Generate animations and turntable renders
- Create batch processing workflows

## Difficulty Level
**Intermediate to Advanced** - Requires Python proficiency and 3D graphics understanding

## Technical Stack
- **3D Software**: Blender 3.6+ (with Python 3.10+)
- **Scripting**: Python 3.10+, Blender Python API (bpy)
- **Rendering**: Cycles (ray tracing) or EEVEE (real-time)
- **Materials**: Shader nodes, PBR materials
- **Add-ons**:
  - Archimesh (architecture helper)
  - Node Wrangler
  - Import-Export: glTF, OBJ
- **Libraries**: NumPy, Pillow, matplotlib (for data visualization)

## Room Specifications
- **Dimensions**: 2.83m × 2.75m × 2.50m
- **Units**: Metric (meters)
- **Color Palettes**: Programmatically applied Feng Shui colors
- **Furniture**: Procedurally placed or imported models
- **Output**: 4K renders, 360° turntables, animations

## Requirements

### 1. Automated Room Generation
- [x] Programmatically create room geometry
- [x] Generate walls, floor, ceiling with correct dimensions
- [x] Add windows and doors
- [x] Apply materials with shader nodes
- [x] Set up camera and lighting

### 2. Furniture System
- [x] Import 3D models (GLTF, FBX, OBJ)
- [x] Programmatically place furniture
- [x] Automatic collision detection
- [x] Randomized variations
- [x] Parametric furniture generation

### 3. Material Library
- [x] PBR material system
- [x] Wood textures (oak, walnut, pine)
- [x] Paint finishes (matte, satin, gloss)
- [x] Fabric materials
- [x] Metal materials

### 4. Lighting & Rendering
- [x] HDRI environment lighting
- [x] Artificial light sources
- [x] Sun lamps for natural light
- [x] Render settings optimization
- [x] Denoising

### 5. Batch Processing
- [x] Generate multiple design variations
- [x] Render from multiple angles
- [x] Create 360° turntable animations
- [x] Export scenes in various formats

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Blender comes with embedded Python
# Location varies by OS:
# - Windows: C:\Program Files\Blender Foundation\Blender X.X\X.X\python\bin\python.exe
# - macOS: /Applications/Blender.app/Contents/Resources/X.X/python/bin/python3.10
# - Linux: /usr/share/blender/X.X/python/bin/python3.10

# Create project directory
mkdir blender-room-designer
cd blender-room-designer

# Create virtual environment (optional, for development)
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install helper libraries
pip install numpy pillow matplotlib

# Project structure
# blender-room-designer/
# ├── scripts/
# │   ├── room_generator.py
# │   ├── furniture_manager.py
# │   ├── material_library.py
# │   └── render_manager.py
# ├── assets/
# │   ├── models/
# │   ├── textures/
# │   └── hdri/
# ├── output/
# └── config/
#     └── room_config.json
```

**File: `config/room_config.json`**
```json
{
  "room": {
    "width": 2.83,
    "depth": 2.75,
    "height": 2.50,
    "wall_thickness": 0.15
  },
  "palettes": {
    "warm": {
      "wall": [0.831, 0.451, 0.369],
      "floor": [0.906, 0.835, 0.769],
      "ceiling": [0.290, 0.247, 0.208]
    },
    "soft": {
      "wall": [0.722, 0.773, 0.839],
      "floor": [0.961, 0.953, 0.937],
      "ceiling": [0.235, 0.247, 0.255]
    },
    "calm": {
      "wall": [0.659, 0.710, 0.627],
      "floor": [0.949, 0.922, 0.851],
      "ceiling": [0.761, 0.773, 0.753]
    }
  },
  "furniture": [
    {
      "type": "desk",
      "position": [-0.8, 0.0, -0.8],
      "rotation": [0, 0, 0],
      "scale": [1.2, 0.75, 0.6]
    },
    {
      "type": "piano",
      "position": [0.5, 0.0, -1.2],
      "rotation": [0, 0, 0],
      "scale": [1.4, 1.2, 0.6]
    },
    {
      "type": "bed",
      "position": [0.8, 0.0, 0.3],
      "rotation": [0, 90, 0],
      "scale": [1.0, 0.4, 2.0]
    }
  ],
  "render": {
    "resolution_x": 3840,
    "resolution_y": 2160,
    "samples": 256,
    "engine": "CYCLES",
    "denoiser": true
  }
}
```

### Step 2: Room Generator Script

**File: `scripts/room_generator.py`**
```python
import bpy
import json
import os
from mathutils import Vector, Euler
from math import radians

class RoomGenerator:
    """Generate 3D room with Blender Python API"""

    def __init__(self, config_path="config/room_config.json"):
        self.config = self.load_config(config_path)
        self.room_data = self.config["room"]
        self.palette = None

    def load_config(self, path):
        """Load configuration from JSON"""
        with open(path, 'r') as f:
            return json.load(f)

    def clear_scene(self):
        """Delete all objects in scene"""
        bpy.ops.object.select_all(action='SELECT')
        bpy.ops.object.delete(use_global=False)

        # Clear orphan data
        for block in bpy.data.meshes:
            if block.users == 0:
                bpy.data.meshes.remove(block)

        for block in bpy.data.materials:
            if block.users == 0:
                bpy.data.materials.remove(block)

    def create_room(self, palette_name="warm"):
        """Create complete room with walls, floor, ceiling"""
        self.clear_scene()
        self.palette = self.config["palettes"][palette_name]

        width = self.room_data["width"]
        depth = self.room_data["depth"]
        height = self.room_data["height"]

        # Create floor
        self.create_floor(width, depth)

        # Create ceiling
        self.create_ceiling(width, depth, height)

        # Create walls
        self.create_walls(width, depth, height)

        # Create window
        self.create_window(width, height)

        # Setup lighting
        self.setup_lighting(width, depth, height)

        # Setup camera
        self.setup_camera(width, depth, height)

        print(f"Room created: {width}m × {depth}m × {height}m")

    def create_floor(self, width, depth):
        """Create floor plane"""
        bpy.ops.mesh.primitive_plane_add(
            size=1,
            location=(0, 0, 0)
        )

        floor = bpy.context.active_object
        floor.name = "Floor"
        floor.scale = (width / 2, depth / 2, 1)

        # Apply material
        floor_mat = self.create_pbr_material(
            "Floor_Material",
            base_color=self.palette["floor"],
            roughness=0.8,
            metallic=0.0
        )
        floor.data.materials.append(floor_mat)

        return floor

    def create_ceiling(self, width, depth, height):
        """Create ceiling plane"""
        bpy.ops.mesh.primitive_plane_add(
            size=1,
            location=(0, 0, height)
        )

        ceiling = bpy.context.active_object
        ceiling.name = "Ceiling"
        ceiling.scale = (width / 2, depth / 2, 1)

        # Apply material
        ceiling_mat = self.create_pbr_material(
            "Ceiling_Material",
            base_color=self.palette["ceiling"],
            roughness=0.9,
            metallic=0.0
        )
        ceiling.data.materials.append(ceiling_mat)

        return ceiling

    def create_walls(self, width, depth, height):
        """Create four walls"""
        wall_thickness = self.room_data["wall_thickness"]

        walls = []

        # Back wall
        back_wall = self.create_wall(
            "Back_Wall",
            width, height, wall_thickness,
            location=(0, -depth/2, height/2),
            rotation=(0, 0, 0)
        )
        walls.append(back_wall)

        # Front wall
        front_wall = self.create_wall(
            "Front_Wall",
            width, height, wall_thickness,
            location=(0, depth/2, height/2),
            rotation=(0, 0, 0)
        )
        walls.append(front_wall)

        # Left wall
        left_wall = self.create_wall(
            "Left_Wall",
            depth, height, wall_thickness,
            location=(-width/2, 0, height/2),
            rotation=(0, 0, radians(90))
        )
        walls.append(left_wall)

        # Right wall (with window cutout)
        right_wall = self.create_wall(
            "Right_Wall",
            depth, height, wall_thickness,
            location=(width/2, 0, height/2),
            rotation=(0, 0, radians(90))
        )
        walls.append(right_wall)

        return walls

    def create_wall(self, name, width, height, thickness, location, rotation):
        """Create individual wall"""
        bpy.ops.mesh.primitive_cube_add(
            size=1,
            location=location,
            rotation=rotation
        )

        wall = bpy.context.active_object
        wall.name = name
        wall.scale = (width/2, thickness/2, height/2)

        # Apply material
        wall_mat = self.create_pbr_material(
            f"{name}_Material",
            base_color=self.palette["wall"],
            roughness=0.85,
            metallic=0.0
        )
        wall.data.materials.append(wall_mat)

        return wall

    def create_window(self, room_width, room_height):
        """Create window on right wall"""
        window_width = 0.8
        window_height = 1.2
        window_y = 0

        bpy.ops.mesh.primitive_plane_add(
            size=1,
            location=(room_width/2 - 0.05, window_y, room_height/2),
            rotation=(0, radians(90), 0)
        )

        window = bpy.context.active_object
        window.name = "Window"
        window.scale = (window_width/2, window_height/2, 1)

        # Glass material
        glass_mat = self.create_glass_material("Glass_Material")
        window.data.materials.append(glass_mat)

        return window

    def create_pbr_material(self, name, base_color, roughness, metallic):
        """Create PBR material with shader nodes"""
        mat = bpy.data.materials.new(name=name)
        mat.use_nodes = True

        nodes = mat.node_tree.nodes
        links = mat.node_tree.links

        # Clear default nodes
        nodes.clear()

        # Create nodes
        output_node = nodes.new(type='ShaderNodeOutputMaterial')
        output_node.location = (300, 0)

        bsdf_node = nodes.new(type='ShaderNodeBsdfPrincipled')
        bsdf_node.location = (0, 0)

        # Set properties
        bsdf_node.inputs['Base Color'].default_value = (*base_color, 1.0)
        bsdf_node.inputs['Roughness'].default_value = roughness
        bsdf_node.inputs['Metallic'].default_value = metallic

        # Link nodes
        links.new(bsdf_node.outputs['BSDF'], output_node.inputs['Surface'])

        return mat

    def create_glass_material(self, name):
        """Create glass material for window"""
        mat = bpy.data.materials.new(name=name)
        mat.use_nodes = True

        nodes = mat.node_tree.nodes
        links = mat.node_tree.links

        nodes.clear()

        output_node = nodes.new(type='ShaderNodeOutputMaterial')
        output_node.location = (300, 0)

        glass_node = nodes.new(type='ShaderNodeBsdfGlass')
        glass_node.location = (0, 0)
        glass_node.inputs['Color'].default_value = (0.8, 0.9, 1.0, 1.0)
        glass_node.inputs['Roughness'].default_value = 0.1
        glass_node.inputs['IOR'].default_value = 1.45

        links.new(glass_node.outputs['BSDF'], output_node.inputs['Surface'])

        return mat

    def setup_lighting(self, width, depth, height):
        """Setup scene lighting"""

        # Sun light (natural light through window)
        bpy.ops.object.light_add(
            type='SUN',
            location=(width/2, 0, height + 2)
        )
        sun = bpy.context.active_object
        sun.name = "Sun"
        sun.data.energy = 3.0
        sun.rotation_euler = Euler((radians(45), 0, radians(45)), 'XYZ')

        # Area light (ceiling light)
        bpy.ops.object.light_add(
            type='AREA',
            location=(0, 0, height - 0.1)
        )
        ceiling_light = bpy.context.active_object
        ceiling_light.name = "Ceiling_Light"
        ceiling_light.data.energy = 100
        ceiling_light.data.size = 0.5
        ceiling_light.data.color = (1.0, 0.95, 0.85)

        # Point light (desk lamp)
        bpy.ops.object.light_add(
            type='POINT',
            location=(-0.8, -0.8, 1.0)
        )
        desk_light = bpy.context.active_object
        desk_light.name = "Desk_Light"
        desk_light.data.energy = 50
        desk_light.data.color = (1.0, 0.9, 0.7)

        print("Lighting setup complete")

    def setup_camera(self, width, depth, height):
        """Setup camera for rendering"""
        camera_distance = 4.0

        bpy.ops.object.camera_add(
            location=(width/2 + 2, depth/2 + 2, height/2 + 1)
        )

        camera = bpy.context.active_object
        camera.name = "Camera"

        # Point camera at room center
        direction = Vector((0, 0, height/2)) - camera.location
        camera.rotation_euler = direction.to_track_quat('Z', 'Y').to_euler()

        # Set as active camera
        bpy.context.scene.camera = camera

        # Set camera properties
        camera.data.lens = 35
        camera.data.sensor_width = 36

        print("Camera setup complete")

    def add_furniture(self, furniture_config):
        """Add furniture to room"""
        for item in furniture_config:
            self.create_furniture(
                item["type"],
                item["position"],
                item["rotation"],
                item["scale"]
            )

    def create_furniture(self, ftype, position, rotation, scale):
        """Create simple furniture (can be replaced with imports)"""

        if ftype == "desk":
            self.create_desk(position, rotation, scale)
        elif ftype == "piano":
            self.create_piano(position, rotation, scale)
        elif ftype == "bed":
            self.create_bed(position, rotation, scale)

    def create_desk(self, position, rotation, scale):
        """Create simple desk"""
        # Desktop
        bpy.ops.mesh.primitive_cube_add(
            size=1,
            location=(position[0], position[1], position[2] + scale[1]/2)
        )
        desktop = bpy.context.active_object
        desktop.name = "Desk_Top"
        desktop.scale = (scale[0]/2, scale[2]/2, scale[1]/10)

        # Apply wood material
        wood_mat = self.create_pbr_material(
            "Wood_Desk",
            base_color=(0.545, 0.435, 0.278),
            roughness=0.7,
            metallic=0.0
        )
        desktop.data.materials.append(wood_mat)

        return desktop

    def create_piano(self, position, rotation, scale):
        """Create simple piano"""
        bpy.ops.mesh.primitive_cube_add(
            size=1,
            location=(position[0], position[1], position[2] + scale[1]/2)
        )
        piano = bpy.context.active_object
        piano.name = "Piano"
        piano.scale = (scale[0]/2, scale[2]/2, scale[1]/2)

        # Dark wood material
        piano_mat = self.create_pbr_material(
            "Piano_Material",
            base_color=(0.172, 0.141, 0.086),
            roughness=0.3,
            metallic=0.0
        )
        piano.data.materials.append(piano_mat)

        return piano

    def create_bed(self, position, rotation, scale):
        """Create simple bed"""
        bpy.ops.mesh.primitive_cube_add(
            size=1,
            location=(position[0], position[1], position[2] + scale[1]/2)
        )
        bed = bpy.context.active_object
        bed.name = "Bed"
        bed.scale = (scale[0]/2, scale[2]/2, scale[1]/2)
        bed.rotation_euler = Euler((0, 0, radians(rotation[1])), 'XYZ')

        # Fabric material
        fabric_mat = self.create_pbr_material(
            "Fabric_Material",
            base_color=(0.910, 0.910, 0.910),
            roughness=0.9,
            metallic=0.0
        )
        bed.data.materials.append(fabric_mat)

        return bed


def main():
    """Main execution"""
    generator = RoomGenerator()

    # Create room with warm palette
    generator.create_room(palette_name="warm")

    # Add furniture
    generator.add_furniture(generator.config["furniture"])

    print("Room generation complete!")


if __name__ == "__main__":
    main()
```

### Step 3: Render Manager

**File: `scripts/render_manager.py`**
```python
import bpy
import os

class RenderManager:
    """Manage rendering settings and batch renders"""

    def __init__(self, output_dir="output"):
        self.output_dir = output_dir
        os.makedirs(output_dir, exist_ok=True)

    def configure_render_settings(self, config):
        """Configure Blender render settings"""
        scene = bpy.context.scene
        render = scene.render

        # Resolution
        render.resolution_x = config.get("resolution_x", 1920)
        render.resolution_y = config.get("resolution_y", 1080)
        render.resolution_percentage = 100

        # Engine
        engine = config.get("engine", "CYCLES")
        scene.render.engine = engine

        if engine == "CYCLES":
            scene.cycles.samples = config.get("samples", 128)
            scene.cycles.use_denoising = config.get("denoiser", True)
            scene.cycles.denoiser = 'OPENIMAGEDENOISE'

            # Use GPU if available
            cycles_prefs = bpy.context.preferences.addons['cycles'].preferences
            cycles_prefs.compute_device_type = 'CUDA'
            scene.cycles.device = 'GPU'

        # Output format
        render.image_settings.file_format = 'PNG'
        render.image_settings.color_mode = 'RGBA'

        print(f"Render settings configured: {render.resolution_x}x{render.resolution_y}, {engine}")

    def render_image(self, output_name):
        """Render single image"""
        filepath = os.path.join(self.output_dir, output_name)
        bpy.context.scene.render.filepath = filepath
        bpy.ops.render.render(write_still=True)
        print(f"Rendered: {filepath}")

    def render_turntable(self, num_frames=120, output_prefix="turntable"):
        """Render 360° turntable animation"""
        scene = bpy.context.scene
        camera = scene.camera

        scene.frame_start = 1
        scene.frame_end = num_frames

        # Animate camera rotation
        for frame in range(1, num_frames + 1):
            scene.frame_set(frame)
            angle = (frame / num_frames) * 360
            camera.rotation_euler.z = radians(angle)
            camera.keyframe_insert(data_path="rotation_euler", index=2)

        # Render animation
        scene.render.filepath = os.path.join(self.output_dir, output_prefix)
        bpy.ops.render.render(animation=True)

        print(f"Turntable rendered: {num_frames} frames")

    def render_multiple_angles(self, angles, output_prefix="view"):
        """Render from multiple camera angles"""
        camera = bpy.context.scene.camera
        original_rotation = camera.rotation_euler.copy()

        for i, (x, y, z) in enumerate(angles):
            camera.rotation_euler = Euler((radians(x), radians(y), radians(z)), 'XYZ')
            self.render_image(f"{output_prefix}_{i:02d}.png")

        camera.rotation_euler = original_rotation

from math import radians
from mathutils import Euler

def batch_render_palettes():
    """Render room with all three color palettes"""
    from room_generator import RoomGenerator

    palettes = ["warm", "soft", "calm"]
    render_mgr = RenderManager(output_dir="output/palettes")

    for palette in palettes:
        gen = RoomGenerator()
        gen.create_room(palette_name=palette)
        gen.add_furniture(gen.config["furniture"])

        render_mgr.configure_render_settings(gen.config["render"])
        render_mgr.render_image(f"room_{palette}.png")


if __name__ == "__main__":
    batch_render_palettes()
```

### Step 4: Running the Script in Blender

```bash
# Command line rendering
blender --background --python scripts/room_generator.py

# With custom output
blender --background --python scripts/room_generator.py -- --output=output/room.png

# Render all palettes
blender --background --python scripts/render_manager.py
```

**File: `run_blender.sh`** (Helper script)
```bash
#!/bin/bash
# Run Blender script

BLENDER_PATH="/Applications/Blender.app/Contents/MacOS/Blender"  # Adjust for your OS
SCRIPT_PATH="scripts/room_generator.py"

$BLENDER_PATH --background --python $SCRIPT_PATH

echo "Blender script execution complete"
```

## Expected Outputs

1. **3D Room Model**: Accurate dimensions with walls, floor, ceiling
2. **Furnished Scene**: Desk, piano, bed placed programmatically
3. **Material System**: PBR materials with proper colors
4. **High-Quality Renders**: 4K resolution images
5. **Multiple Views**: Batch renders from different angles
6. **Turntable Animation**: 360° rotation video

## Bonus Challenges

1. **Procedural Textures**: Generate wood grain, fabric patterns
2. **Advanced Lighting**: HDREnvironment maps, volumetrics
3. **Animation**: Walk-through camera paths
4. **Parametric Furniture**: Generate furniture from parameters
5. **Material Variations**: Auto-generate material swatches
6. **Photogrammetry**: Import scanned real-world textures
7. **AI Integration**: Use Blender with Stable Diffusion

## Resources

- [Blender Python API Docs](https://docs.blender.org/api/current/)
- [Blender Scripting Guide](https://docs.blender.org/manual/en/latest/advanced/scripting/index.html)
- [CG Cookie Blender Python](https://cgcookie.com/courses/blender-python)
- [Blender Artists Forum](https://blenderartists.org/)

## Success Criteria

- [ ] Room generates with exact dimensions
- [ ] All three palettes render correctly
- [ ] Furniture places without collisions
- [ ] Materials look realistic
- [ ] Renders complete without errors
- [ ] 4K output at 60 seconds or less
- [ ] Batch processing works for all variations
- [ ] Scripts can be run headlessly (no GUI)
