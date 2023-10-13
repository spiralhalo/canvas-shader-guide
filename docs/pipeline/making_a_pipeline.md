# Making a Pipeline

To get started, let's create a basic Pipeline.

Start by creating an empty resource pack. This will be the blank canvas for our Pipeline.

***Note: everything prefixed with "my_" implies any custom name will work.***

```
.minecraft/resourcepacks/my_pack/
```

Create the assets folder and a namespace folder inside it:

```
.minecraft/resourcepacks/my_pack/assets/
.minecraft/resourcepacks/my_pack/assets/my_namespace/
```

Create a pipelines folder inside the namespace folder:

```
.minecraft/resourcepacks/my_pack/assets/my_namespace/pipelines/
```

Then inside the pipelines folder, create your Pipeline **json5** file:

```
.minecraft/resourcepacks/my_pack/assets/my_namespace/pipelines/my_pipeline.json5
```

Then, before creating the Pipeline file, generally you want to decide whether to enable Fabulous mode or not.

| Mode | Description |
| -- | -- |
| without Fabulous mode | Faster; All render targets the solid buffer; No composite pass. |
| with Fabulous mode | Slightly slower; Separate buffer for each target; Fabulous composite pass. |

## Basic Pipeline

Since Canvas ships with a non-fabulous Pipeline called "Canvas Basic", we'll call non-fabulous Pipelines "basic" Pipelines.

To create a basic pipeline, insert the following into your Pipeline json5 file:

## Fabulous Pipeline

## Language files
