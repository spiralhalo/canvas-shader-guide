# The Shadow Map

The shadow map is the image that stores the depth from the sun's perspective. By comparing the depth from the sun's point of view with the depth from the player's point of view, we can tell whether there are shadows. Since this requires rendering the entire scene a second time, it's significantly slower than not using shadows at all.


## The boring JSON side

To start, you need to create the shadow configuration JSON file. You can create this directly in your pipeline description file, or you can use another file and `include` the file in your pipeline description file.

Let's assume you're doing the latter, and create `my_shadow_config.json5`. This can be done anywhere under the `assets` folder. See comments for specific parameter descriptions.

```json5
{
	images: [
		{
			name: "my_shadow_map",
            // Size can be whatever you want, but it's recommended to use a power of 2, like 1024
			size: 1024,
			internalFormat: "DEPTH_COMPONENT32",
			pixelFormat: "DEPTH_COMPONENT",
			pixelDataType: "FLOAT",
			target: "TEXTURE_2D_ARRAY",
			depth: 4,
			texParams: [
				{name: "TEXTURE_MIN_FILTER", val: "LINEAR"},
				{name: "TEXTURE_MAG_FILTER", val: "LINEAR"},
				{name: "TEXTURE_WRAP_S", val: "CLAMP_TO_EDGE"},
				{name: "TEXTURE_WRAP_T", val: "CLAMP_TO_EDGE"},
				{name: "TEXTURE_COMPARE_MODE", val: "COMPARE_REF_TO_TEXTURE"},
				{name: "TEXTURE_COMPARE_FUNC", val: "LEQUAL"}
			]
		}
	],

	framebuffers: [
		{
			name: "my_shadow_framebuffer",
			// The shadow pass only writes to depth, so we provide a depth attachment
			depthAttachment: {
				image: "my_shadow_map",
				clearDepth: 1.0
			},
		}
	],
	
	skyShadows: {
		framebuffer: "my_shadow_framebuffer",
		// These are up to your personal preference
		allowEntities: true,
		allowParticles: true,
		supportForwardRender: true,
		// You can name the shaders whatever you want, and place them
		// wherever you want, but make sure you specify the correct path 
		// here, relative to your assets/namespace/... folder
		vertexSource: "my_pipeline_namespace:shaders/.../shadow.vert",
		fragmentSource: "my_pipeline_namespace:shaders/.../shadow.frag",
		// Parameters to glPolygonOffset(offsetSlopeFactor, offsetBiasUnits)
		offsetSlopeFactor: 1.1,
		offsetBiasUnits: 4.0,
		// Cascade radii are up to personal preference. 
		// Aim for a balance between detail up close and detail far away.
		// 
		// There are four cascades, notice that only three numbers are
		// supplied here.
		// This is because the 0th cascade always covers the largest
		// allowed shadow render distance.
		cascadeRadius: [96, 32, 12]
	},

	sky: {
		// The angle of the sun. 0 means the sun follows the same 
		// trajectory as vanilla. Negative numbers are allowed.
		defaultZenithAngle: 40
	}
}
```

## The fun shader part

Now that we have the pipeline shadow descriptor done, we can move on to writing the actual shadow shaders. Both the vertex and fragment shaders are surprisingly simple, because you're not doing any fancy shading.

The shadow program will be run multiple times, once for each cascade.

### The vertex shader

Navigate to where you created your vertex shader. If you're wondering where a good place is, I like them in the same directory as the material program shaders, or the gbuffer shaders: `assets/my_namespace/shaders/gbuffer/shadow.vert`

```glsl
#include frex:shaders/api/vertex.glsl
#include frex:shaders/api/view.glsl

uniform int frxu_cascade;

void frx_pipelineVertex() {
	// Move from model space to camera space
	frx_vertex += frx_modelToCamera;

	// Multiply for the shadow projection with the current cascade
	gl_Position = frx_shadowViewProjectionMatrix(frxu_cascade) * frx_vertex;
}
```

As you can see, this shader is very simple; all we need to do is transform vertices to the sun's point of view.

### The fragment shader


