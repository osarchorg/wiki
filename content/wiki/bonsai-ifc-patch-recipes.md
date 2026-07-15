---
title: "IFC Patch Recipes"
url: "/bonsai-ifc-patch-recipes/"
parent: "/bonsai/"
aliases: ["/BlenderBIM_Add-on/IFC_Patch_Recipes/", "/blenderbim-add-on-ifc-patch-recipes/"]
categories: []
lastmod: "2026-07-15T00:00:00Z"
---

Ifc Patch can be found in the Scene Properties Tab,
Under the IFC Quality Control Drop down menu

Code for IFC Patch Recipes in located [here](https://github.com/IfcOpenShell/IfcOpenShell/tree/v0.8.0/src/ifcpatch/ifcpatch/recipes).

For Windows use <code>\</code>, for Mac and GNU/Linux use <code>/</code> in the path


## MergeDuplicateTypes
Merges duplicate element types into a single type. Revit in particular tends to export many duplicate types (for example mirrored doors, columns, or certain MEP equipment), so a door schedule you expect to have 3 types might actually store 6 or more. Occurrences of the removed types are remapped to the surviving type.

Types are grouped by an attribute. There are two arguments:

- **attribute** (default <code>Tag</code>) — the attribute used to decide what counts as a duplicate. Revit stores its internal Element ID in <code>Tag</code>, so two types sharing a Tag are duplicates. You can instead use <code>Name</code> if users made multiple same-named types as a workaround.
- **should_merge_null** (default <code>False</code>) — whether types that have an *empty* value for the chosen attribute should all be merged together.

Only types of the **same IFC class** are ever merged, so unrelated types (e.g. an annotation type and a beam type) are never combined.

**Note on should_merge_null:** leave this off (the default) unless you know what you are doing. An empty attribute is not evidence of duplication — turning this on merges *every* untagged type of a given class into one, which can silently destroy distinct, differently-named types (a common problem on Bonsai-authored or mixed models). Genuine Revit duplicates always carry a populated <code>Tag</code>, so the default is safe for that workflow. Enable it explicitly with arguments <code>["Tag", true]</code> only when you truly want all untagged types collapsed.

## MergeProject
An example, for Windows:

{{< wiki-image src="/media/merge-project3.png" alt="MergeProject3.png" mode="inline" >}}

## OffsetObjectPlacements
The arguments for OffsetObjectPlacements are a list of numbers. If you specify 3 numbers [X,Y,Z], the coordinates will be offset by X, Y, Z. If you specify 4 numbers [X,Y,Z,Az], it will be offset by X, Y, Z and rotated along the Z axis by Az. If you specify 6 numbers [X,Y,Z,Ax,Ay,Az] it will translate along all three axes and also rotate along all three axes. [-source](https://community.osarch.org/discussion/comment/12408/#Comment_12408)

## RecycleNonRootedElements
Consolidates redundant non-rooted entities, like the following example, down to one entity.

<code>#45=IFCOWNERHISTORY(#9,#8,.READWRITE.,.MODIFIED.,1629040293,#9,#8,1629040293);</code>
<code>#52=IFCOWNERHISTORY(#9,#8,.READWRITE.,.MODIFIED.,1629040293,#9,#8,1629040293);</code>

{{< wiki-image src="/media/recycle-non-rooted-elements.png" alt="RecycleNonRootedElements.png" mode="inline" >}}
