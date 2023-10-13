# What is a Pipeline?

Canvas Pipeline is a configurable set of shader program and passes with the capability of shading the full game as well as GUI items. Additionally, a Pipeline also enables specific rendering features such as shadows, post-processing, and PBR (work in progress).

## What is a material shader?

Game objects like terrain, entities, and particles are rendered with materials. Therefore the program used to render them is known as the Material Program.

The Material Program is controlled by the Pipeline, but it may be extended by mods and resource packs via material shaders. To put it simply, material shaders control the early part of object rendering before it's being passed over to the Pipeline.

There is also the depth-pass shader, which works like material shader but for shadow pass.

For more details, see the Material Program and Material Shader sections. (TODO)

## What is a shader pack?

In the past, custom Minecraft shaders are distributed as .zip files called "shader pack" (or simply "shader"). This holds true to this day.

However, Canvas shaders are distributed as **resource packs**. These resource packs may contain:
- One or more Pipeline(s),
- Material shaders,
- Material maps,
- Language files, textures, and other common resources.

As Canvas packs contain shaders, it makes sense to call them "shader pack" as well.
