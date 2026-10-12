---
recording_id: vdqbtAYK
duration: 2m38s
created_at: 2026-08-17
updated_at: 2026-08-17
#Godot
---
# Breath of the Wild-style windy grass shader using vertex colors and noise

Date: 2026-08-17

## Summary

After completing a minimal boilerplate for basic controller schemes, the focus shifts to creating Breath of the Wild–style windy grass as an early proof of concept. The approach likely involves alpha-blended grass textures (or possibly meshes) and vertex coloring in Blender so vertex color intensity corresponds to height above the mesh base. In the shader, this height-based value would control how strongly wind affects different parts of the grass. Wind animation could start with simple sine-based motion and progress to more complex patterns like curl noise for more natural variation. Another key task is finding a way to color or shade the underlying terrain so it blends seamlessly with the grass, reducing the visible separation between ground and grass, likely by researching established techniques.

## Transcript

After I've kind of gotten the absolute bare-bones boilerplate design for basic controller schemes, I'd like to really focus on figuring out how to do windy grass along the lines of Breath of the Wild. I already know that that encompasses a few key points, a few key things. Alpha blended textures of grass that are, or perhaps meshes, but I think it's textures, I'm not 100% sure, I have to look this up. 

But within something like Blender, their vertices are actually colored so that the color changes depending on how much higher from the base of the local coordinate system of that mesh the vertice is. And in that way, when you're looking at it through a shader program, you can see, you can use the color to measure the color's intensity equates to its height, and you can use that to figure out how much more it's affected by wind. 

From there, you can go from maybe a basic sine wave or a basic two-dimensional sine wave or sinc function to something like curl noise that sends really weird patterns across. And being able to handle that is, and make it look true to life. Like another thing I'm going to have to figure out is how to set the bottom, how to color the bottom terrain layer so that it seemingly blends in with the grass and it's harder to notice that the grass is this removed texture. 

And so I'm going to have to look up techniques for that, most likely by discussing it with AI. But that's one goal of mine that I would really like to achieve early on, just as a proof of concept.

## Tags

#Godot

# Related notes

- [[2026-08-17 Godot skeleton repos for first- and third-person controller boilerplate]]