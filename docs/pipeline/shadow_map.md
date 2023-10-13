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

Create the fragment shader in the path you decided upon earlier. 

```glsl
#include frex:shaders/api/material.glsl
#include frex:shaders/api/fragment.glsl

void frx_pipelineFragment() {
  gl_FragDepth = gl_FragCoord.z;
}
```

This is even simpler than the vertex shader. Since there are no color attachments, all we need to do is write to depth.

Now, if you reload your pipeline, you should be at a point where the shadow map is rendered properly! However, we want to use the shadow map to render shadows in our lighting system.

### Sampling shadows

This section will be a little abstract, because shadow rendering can go from being very simple to very complicated, and there are a bunch of different moving parts.

To be able to sample shadows, we need to first set up the shadow positions using some fancy transformations. 

> Note: If you decide to do these transformations in the vertex shader, keep in mind that beyond 12-16 chunks render distance, there are likely to be more vertices than pixels on the screen, making your code slower.

First, copy these helper functions into your code. They're annoying to write and give us the information we need regarding cascades, etc.

```glsl
vec3 shadowDist(int cascade, vec4 pos) {
	vec4 c = frx_shadowCenter(cascade);
	return abs((c.xyz - pos.xyz) / c.w);
}

// Function for obtaining the cascade level
int selectShadowCascade(vec4 shadowViewSpacePos) {
	vec3 d3 = shadowDist(3, shadowViewSpacePos);
	vec3 d2 = shadowDist(2, shadowViewSpacePos);
	vec3 d1 = shadowDist(1, shadowViewSpacePos);

	if(all(lessThan(d3, vec3(1.0)))) { return 3; }
	if(all(lessThan(d2, vec3(1.0)))) { return 2; }
	if(all(lessThan(d1, vec3(1.0)))) { return 1; }

	return 0;
}
```

Now, let's set up the shadow position itself. In the code below, `cameraSpacePos` refers to the position in the same space as `frx_vertex.xyz`. If you are in the material program, you can use `frx_vertex.xyz`, otherwise you will need to set up this position yourself using matrix transformations.

```glsl
vec4 shadowViewPos = frx_shadowViewMatrix * vec4(cameraSpacePos, 1.0);
int cascade = selectShadowCascade(shadowViewPos);

vec4 shadowClipPos = frx_shadowProjectionMatrix(cascade) * shadowViewPos;
vec3 shadowScreenPos = (shadowClipPos.xyz / shadowClipPos.w) * 0.5 + 0.5;
```

Now that we have the shadow screen position, we can sample shadows.

> Another note: it's normal to be confused about the shadow transformations. They're pretty unintuitive. To keep it simple, `shadowViewPos` is the actual world-scaled position of the world from the sun's perspective, but with the origin and axes relative to the camera. `shadowClipPos` is the transformation of `shadowViewPos` into device coordinates so that geometry clipping can happen. `shadowScreenPos` is the screen-relative coordinates of the shadow map, and it's what we can use to sample the image. Think of it as texture coordinates, or texcoords for the shadow map.

If you want to sample your shadow map in the material program:

```glsl
float shadowFactor = texture(frxs_shadowMap, vec4(shadowScreenPos.xy, cascade, shadowScreenPos.z));
float rawShadowDepth = texture(frxs_shadowMapTexture, vec3(shadowScreenPos.xy, cascade)).r;
```

If you want to sample your shadow map in a pass shader or fullscreen pass, you will need to set up the sampler manually, using the name of the shadow map image you created. If you want to get the raw shadow factor, you should declare the sampler as `sampler2DArrayShadow`. If you want raw depth, declare the sampler as `sampler2DArray`. If you want both, set up two different samplers that point to the same image.

```glsl
// In the global scope
uniform sampler2DArrayShadow u_myShadowMap;
uniform sampler2DArray u_myShadowMapTexture;

// In the place where you want factors
float shadowFactor = texture(u_myShadowMap, vec4(shadowScreenPos.xy, cascade, shadowScreenPos.z));
float rawShadowDepth = texture(u_myShadowMapTexture, vec3(shadowScreenPos.xy, cascade)).r;
```

Use the shadow factor you get from sampling the shadow map for whatever purpose you want! 

You can also sample the shadow map multiple times with tiny offsets to "blur" the shadows, causing less artifacts when moving. When you are comfortable with shadow rendering, you can look into more advanced shadow sampling techniques, like variable penumbra shadows, which adjusts the blur amount based on how far the shadow caster is to the ground.

As a final note, you will notice that the back faces of blocks still let a bit of sunlight through. This is because simply applying the shadow factor to your lighting isn't exactly realistic - you should multiply by the dot product of the normal vector and the light vector. This multiplication both makes your lighting look more realistic and makes sure that the back faces of blocks are properly in shadow.