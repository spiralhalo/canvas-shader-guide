# Colored Lights (WIP)
This is a feature in Canvas that adds **client-side** colored lighting using the flood-fill algorithm. The data is firstly and priorly absorbed from the block light API, then the item light API, and last fallback being the vanilla light level.

The algorithm works with light propagation hooking to a region (chunk) building for block updates, which was already optimized through occlusion culling. This ensures minimal memory usage.

The actual propagation and texture uploads are done in the Render Thread. Only queued regions are updated to minimize impact on frame time.
# The API
## Block Light API
This is expressed with json files that contain color *(rgb)* data and light level data *(intensity)*.

With two root objects, you can specify the default light values (for the normal default block state) using the `"defaultLight"` object, and the variants (for each unique block state) light values using the `"variants"` object. 

Inside those objects, you can express the RGB color data with `"red":`, `"green":` and `"blue":"`. The intensity is expressed through `lightLevel":` within the **0-15** range.

The fields are optional and may fallback to the fields in `"defaultLight"` if not present, and if there is no `"defaultlight"` object, it will simply fall back to the values that are **server-side**.

These files must be contained in the following paths accordingly: `namespace:lights/block`, `namespace:lights/fluid`.

The API is designed based on both **Material Map** and **Item Light APIs**.

Example json:
```json5
{
  "defaultLight": {
    "red": 0.2,
    "green": 0.2,
    "blue": 0.2
  },
  "variants": {
    "color=red": {
      "lightLevel": 14.0,
      "red": 1.0
    },
    "color=green": {
      "lightLevel": 14.0,
      "green": 1.0
    },
    "color=blue": {
      "lightLevel": 14.0,
      "blue": 1.0
    }
  }
}

```

**Note** that due to this feature being client-side, it won't affect server-side light values. To achieve harmony between visuals and gameplay, mods would require synchronizing server-side light values via other means. Care needs to be taken as [luminance](https://en.wikipedia.org/wiki/Relative_luminance) is perceived differently in colored lights than vanilla lighting.

## Pipeline API
This feature is not recognized by default. To ensure your pipeline has the colored lighting feature on, you must include it in the json file of the pipeline as an object like so:
```json5
  coloredLights: {
    useOcclusionData: false
  }
```

## Shader API
To take advantage of this feature in your shader files, you can call it using specific includes (depending on what functions you want, and what type of shaders you are using the feature on).

### Material & Pipeline Write Shaders
```glsl
#ifdef COLORED_LIGHTS_ENABLED

uniform sampler2D frxs_lightData;

vec4 frx_getLightFiltered(worldPos);
vec4 frx_getLightRaw(worldPos);
vec3 frx_getLight(worldPos, fallback);

#endif
```
You get a predefined sampler of the light data, with functions that are explained below.


### Pipeline Pass Shaders

Including the path `frex:shaders/api/light.glsl` to your shader program, you gain access to multiple functions like:

```glsl
vec4 frx_getLightFiltered(sampler2D lightSampler, vec3 worldPos);

vec4 frx_getLightRaw(sampler2D lightSampler, vec3 worldPos);

vec3 frx_getLight(sampler2D lightSampler, vec3 worldPos, vec3 fallback);

bool frx_lightDataExists(sampler2D lightSampler, vec3 worldPos);
```

`frx_getLightFiltered` Function returns the light color with filtering on.

`frx_getLightRaw` Function returns the light color with no filtering, meaning the raw color (per voxel).

`frx_getLight` Function returns the light color with filtering, and a fallback where there is no light data. (Likely, you can use the vanilla lightmap for fallback (`frx_fragLight.x`)

`frx_lightDataExists` Function checks whether or not the light data is uploaded for a specific region in the world

**Note** that you may need to offset the worldPos by the world normals by a slight amount, `worldPos + frx_vertexNormal.xyz * 0.05` works out.


#### Occlusion Data
This feature contains occlusion data, and you can define whether or not u want it on in the pipeline json object like so:
```json5
  coloredLights: {
    useOcclusionData: true
  }
```

Occlusion data lets you gain access to a function of a variety of data about the light occlusion.

```glsl
struct frx_LightData {
	vec4 light;
	bool isLightSource;
	bool isOccluder;
	bool isFullCube;
};

frx_LightData frx_getLightOcclusionData(sampler2D lightSampler, vec3 worldPos);
```
With a struct, you get the:

`light` unfiltered (raw) light color.

`isLightSource` Determinator for whether a given pixel is a pixel of a light source or not.

`isOccluder` Determinator for whether a given pixel is a pixel of an occluder or not.

`isFullCube` And a determinator for whether a given pixel is a pixel of a full cube or not (regular block unlike a slab or a carpet).

`frx_getLightOcclusionData` Function can give you all that data with a given light sampler, in this case, you may pass `"frex:textures/auto/colored_lights"` to a `sampler2D` to access the desired data.

**Additionally, you may want to watch out for the possibility of shifting the `worldPos` to a slight negative offset instead of a positive normal offset. 
Say the `worldPos` has no offset applied, you may run into cases where you benefit adding a negative offset like so: `worldPos - frx_vertexNormal.xyz * 0.01`.**
