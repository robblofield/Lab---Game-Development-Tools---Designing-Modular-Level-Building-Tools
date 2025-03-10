# Lab - Game Development Tools - Designing Modular Level Building Tools (Part 1)

*In this lab, students will learn how to create a Scriptable Object-based system for configuring level prototyping assets in Unity. This system will allow designers to quickly swap between different prefabs, textures, and colors when building levels.*

This is Part 1 of a three-part series:

1. Setting Up the Prototype System (Today’s Lab)
2. Enhancing the System with Quality-of-Life Features
3. Adding Advanced Features like Rotation Gizmos

Please note the instructions in this lab sheet do not create a working system - It is intentially left with bugs for us to iterate upon as this is typical in how a studio tool is developed over time as the needs for it change with the game.

### **Core Concept: Scriptable Object**  
Unity's implementation of `Scriptable Objects` is a Unity-specific system that uses a commonly adopted programming pattern named th `Prototype Pattern`

The Prototype Patern, builds on Object Oriented Programming (OOP), to create a core class script/object `Prototype` that is capable of being many different flavours, if configured as such.

i.e - Enemy Prototype/Scriptable Object
- Enemy ID - Empty
- Enemy Prefab - Empty
- Enemy AI - Empty
- Enemy HP - Empty
- Enemy DPS - Empty
- Enemy Speed - Empty

Can be configured to:
- Enemy ID - String `<Patrolling Grunt>`
- Enemy Prefab - Grunt_Basic.prefab
- Enemy AI - Enemy_Patrol.cs
- Enemy HP - 200
- Enemy DPS - 3.2
- Enemy Speed - 1.4

Which in turn can be reconfigured for an entirely different enemy:

- Enemy ID - String `<Upgraded Grunt>`
- Enemy Prefab - Grunt_Upgraded.prefab
- Enemy AI - Enemy_SeekAndDestroy.cs
- Enemy HP - 300
- Enemy DPS - 4.5
- Enemy Speed - 2

This type of programming pattern or `architecture` in our code allows us to build out systems/tools as core ideas, allowing us to re-use them as a template for variants very easily.

Once a new scriptable object is created in Unity, you can make a new asset from it by right clicking in the assets folder and creating one (Create > Scriptable Object > `EnemyType`). Then configuring it's set-up in the inspector using the fields/options you have made available in the base template.

### Additional Notes/Documentation
* [Unity Documentation - Manual - Scriptable Object](https://docs.unity3d.com/6000.0/Documentation/Manual/class-ScriptableObject.html)
* [Game Programming Patterns - Chapter - Prototype](https://gameprogrammingpatterns.com/prototype.html)

---

## Section 1 - Brainstorming the Problem

Most custom systems or editor tools we build in a game engine are born through a need to make working and iterating on the project more efficient, easily understandable or open to a wider skillset. Typically this might be developing useful tools that would allow designers to create new functionality in the game without a direct need to know a great deal of coding, or it can be to speed up a slow/monotonous process, allowing for more care and attention to be spent of building good gameplay/levels etc.

Let's look at 3D level building in Unity and the tools available to us. Specifically thinking about trying to maintain modular asset size for rigid style game designs such as grid-based layouts etc, and for making sure that levels designed between multiple people on the team can be consistent in their approach.

- Custom 3D Models in Blender (Pros/Cons)
- Creating Basic Cubes etc in Unity (Pros/Cons)
- Pro-Builder Unity Add-On (Pros/Cons)

How easy is it to keep to a set dimension/scale?
How easy is to to perfectly snap objects together?
How easy is it to swap out items we no longer want?
How easy is it to scale/rotate/position the items?
How easy is it to test the level design in-engine?

Create a list of the features you would want to see in a modular level building tool for a rigid grid-based game. We then rank them into priority so that we can build this iteratively with the core requirements, desirable features, QoL improvements coming at separate stages.

### - Modular Level Building Tool Wishlist
- `Hot-swap pre-set block sizes/shapes (e.g 1x1x1m, 2x3x2m etc)`
- `Hot-swap base texture`
- `Hot-swap base colour`
- Easy duplication
- Easy rotation
- Optional changes via custom drop-down list
- `Runs/Updates in editor not a run time`

Features in this first development phase:
1. Prefab Swapping – Select from predefined core prefabs (e.g., different block sizes).
2. Material Overrides:
    - Color: Choose from a predefined set of colors.
    - Texture: Choose from a predefined set of textures.
3. Live Updates in the Editor (not just runtime).

### Planning the Architecture
`Scriptable Object (PrototypeBlockConfig)`
- Stores predefined prefabs, textures, and colors.
`Prototype Block Component (PrototypeBlock)`
- Allows selection of a prefab from the config list.
- Swaps the prefab in the editor when changed.
- Applies texture and color changes via a MaterialPropertyBlock (non-destructive, per-instance modifications).
`Editor Script (PrototypeBlockEditor)`
- Adds an "Update Block" button in the Unity Inspector for instant updates.

### Step 1. Create the ScriptableObject (PrototypeBlockConfig)
This will hold references to different prefabs, textures, and colors for easy swapping.

Steps:
1. In Unity, go to `Assets/` and create a folder called `PrototypeSystem.`
2. Inside `PrototypeSystem`, create a new C# script and name it `PrototypeBlockConfig.cs.`
3. Open the script and setup the scriptable object:

```csharp
using UnityEngine;
using System.Collections.Generic;

// Create a new menu option in the "CreateAssetMenu". This is what we'll click when we want to make a system that uses this scriptable object.
[CreateAssetMenu(fileName = "NewPrototypeConfig", menuName = "Prototype/BlockConfig")]

// Note that we are not inheriting from "Monobehaviour", instead we are inheriting from "Scriptable Object" - Unity's pre-made implementation of the "prototype pattern"
public class PrototypeBlockConfig : ScriptableObject
{
    // Add in the variables we want to include as configurable options
    public List<GameObject> prefabs;
    public List<Texture> textures;
    public List<Color> colors;
}
```
> Explanation:
> - prefabs → Stores different block prefabs (e.g., 2x2x2m, 2x1x2m).
> - textures → Stores a list of textures to apply.
> - colors → Stores selectable colors for customization.


### Step 2. Create the Prototype Block Component
This script will allow you to swap prefabs, change materials, and update instantly.

Steps:
1. Inside `PrototypeSystem`, create another script: `PrototypeBlock.cs.`
2. Open it and build out the logic

```csharp
using UnityEngine;
using UnityEditor;

[ExecuteInEditMode] // Enables live updates in the editor
public class PrototypeBlock : MonoBehaviour // Note we're back to using monobehaviour again
{
    public PrototypeBlockConfig config; // Reference to the scriptable object base class
    public int selectedPrefabIndex = 0; // The current index value of the array/list for the selected prefab
    public int selectedTextureIndex = 0; // The current index value of the array/list for the selected texture
    public int selectedColorIndex = 0; // The current index value of the array/list for the selected colour

    private Renderer rend; // The render component on the selected prefab
    private MaterialPropertyBlock mpb; // The material property block attached to the render on the selected prefab

    private void OnValidate() // Any time the script is confirmed to be responding
    {
        UpdateBlock(); // Live update when values change
    }

    public void UpdateBlock()
    {
        if (config == null || config.prefabs.Count == 0) return; // If we haven't setup a config reference, or if we havent got any prefabs in our configuration, then don't do anything.

        // Swap Prefab
        GameObject selectedPrefab = config.prefabs[selectedPrefabIndex]; // Set our select prefab equal to the one in the index
        if (selectedPrefab && selectedPrefab != gameObject)
        { // if the selected prefab is not the same as the current gameObject we need to change it
            GameObject newBlock = (GameObject)PrefabUtility.InstantiatePrefab(selectedPrefab, transform.parent);
            newBlock.transform.SetPositionAndRotation(transform.position, transform.rotation);
            newBlock.transform.localScale = transform.localScale;
            newBlock.GetComponent<PrototypeBlock>().config = config; // Set the new block to the configuration
            Undo.RegisterCreatedObjectUndo(newBlock, "Prefab Swap");
            DestroyImmediate(gameObject);// destroy the old block
        }

        // Apply Material Customization
        rend = GetComponent<Renderer>();
        if (rend != null) // If we have a renderer, then do the following
        {
            mpb = new MaterialPropertyBlock();
            rend.GetPropertyBlock(mpb); // Get the material property block so we can update it

            if (config.textures.Count > selectedTextureIndex) // If we have selected a valid texture index (within 0 to the total count of textures we've added)
                mpb.SetTexture("_MainTex", config.textures[selectedTextureIndex]);// Set the input for "_MainTex" to the new texture

            if (config.colors.Count > selectedColorIndex)
                mpb.SetColor("_Color", config.colors[selectedColorIndex]);// Same thing but for colour

            rend.SetPropertyBlock(mpb);// Set the property block after configuring changes
        }
    }
}

```
> Explanation:
> - OnValidate() → Ensures the prefab swaps instantly when changing values in the Inspector.
> - UpdateBlock() →
>   - Swaps the prefab based on the selected option.
>   - Updates the texture & color using MaterialPropertyBlock (to avoid changing shared materials).



### Step 3. Create the Custom Editor Script
This will enhance the Unity Inspector UI, making it easier to swap prefabs, textures, and colors.

Steps:
1. Inside PrototypeSystem, create another script: `PrototypeBlockEditor.cs.`
2. Open it and replace the contents with:

```csharp
using UnityEngine;
using UnityEditor;

[CustomEditor(typeof(PrototypeBlock))] // Custom editor to be used when the object class is `PropertyBlock` - This means it will have different editor behaviours to normal objects.
public class PrototypeBlockEditor : Editor //We are inheriting from editor, as we're making a new custom editor
{
    public override void OnInspectorGUI()
    {
        DrawDefaultInspector();
        // Keep default inspector
        // Then add this custom extra feature - GUI Button "Update Block"
        PrototypeBlock block = (PrototypeBlock)target;
        if (GUILayout.Button("Update Block"))
        {
            block.UpdateBlock();
        }
    }
}

```
> Explanation:
> - Adds an "Update Block" button in the Inspector, allowing manual updates.
> - Uses DrawDefaultInspector() to keep the default fields visible.


### Step 4. Set Up the Prototype System in Unity
Now, we will create assets in Unity for testing.

Steps:
1. Create a `ScriptableObject` Asset
- In Unity, Right-click in the PrototypeSystem folder.
Go to `Create > Prototype > BlockConfig`.
- Rename it to `PrototypeConfig`.
- Open the asset and add:
    - `Prefabs`: Drag in different cube sizes (e.g., 2x2x2m, 2x1x2m, etc.).
    - `Textures`: Drag in a few textures.
    - `Colors`: Add some colors using the + button.

2. Create a Prefab for Testing
- Create a new `empty GameObject` in the scene (GameObject > Create Empty).
- Rename it `PrototypeBlock`.
- Attach the `PrototypeBlock.cs` script to it.
- Assign the `PrototypeConfig` asset to it. **See Note Below**
- Create a basic cube inside it (GameObject > 3D Object > Cube). Or Use Pro-Builder to make your cubes
- Attach a material to the cube (can be a default one).
- Convert it into a prefab (Right-click > Prefab > Create Prefab).

3. Test the System
- Drag the PrototypeBlock prefab into the scene.
- In the Inspector, change the Prefab, Texture, or Color.
- Watch the object update in real-time!

#### N.B - Assigning The Prototype Config Asset
1. Locate the PrototypeConfig Asset
In Unity, open the Project window.
Navigate to the PrototypeSystem folder where you created PrototypeConfig.
2. Select the PrototypeBlock GameObject
In the Hierarchy panel, find the PrototypeBlock GameObject that you created.
Click on it to select it.
3. Assign the ScriptableObject
In the Inspector panel, look for the PrototypeBlock (Script) component.
You should see a field named "Config" (this is the PrototypeBlockConfig reference).
Drag the PrototypeConfig asset from the Project window into the Config field in the Inspector.

## Section 2 - Lab Task - Evaluating The Tool?

We've made our first attempt at outlining the tool and have created the core architecture. Whilst that is sound, it still doesn't function exactly as we would want it to and so we need to debug it, and we need to get these elements working before thinking about new features.

### What are known bugs/issues with this implementation so far?

> when I select an index in the prefab slot it does generate the correct prefab, but in the hierarchy it generates it as a new object that isn't a child of the scriptable object.

Solution: We need to find a way to automatically make that new prefab a child of the core scriptable object.

> If we do get instantiation and parenting working correctly, how will we make sure we select and delete the old one?

Solution: We'll need to keep track of the old one and then destroy it after the new one has been created. We may need to use some debugs to see when a new prefab is being created and whether it is designating it the old or new version. We might be using `illegal` methods when destroying stuff.

### How to resolve these issues?
I would recommend we do some planning and trial/error to see what works/doesn't work. We should use a lot of debugs to try to diagnose the problem. Then we should either dive into documentation to see if we can find a solution --or-- we should present the scripts and our findings from debugs to ChatGPT to ask for advice or possible solutions to explore.


### Final code for today (but seriously don't just skip to here, you won't learn nearly as much)

Here's some working code, but its important to recognise it is not the `correct` or `only` way to build this functionality. It came from me using the trial and error methods, documentation and sanity checking in ChatGPT to build out this solution.

Being able to debug and improve it via this method is critical to understanding it - especially as this is the basis for a large extensible project we hope to add more functionality to.

> N.B - Also note that I have changed a method name here from `block.UpdateBlock();` to `block.SafeUpdateBlock()` so need to update my references to it in other scripts.

```csharp
using UnityEngine;
using UnityEditor;

[ExecuteInEditMode]
public class PrototypeBlock : MonoBehaviour
{
    public PrototypeBlockConfig config;
    public int selectedPrefabIndex = 0;
    public int selectedTextureIndex = 0;
    public int selectedColorIndex = 0;

    private void OnValidate()
    {
        EditorApplication.delayCall += SafeUpdateBlock; // Ensures execution happens after validation
    }

    private void SafeUpdateBlock()
    {
        if (this == null) return; // Prevents execution on deleted objects
        if (config == null || config.prefabs.Count == 0) return;

        Debug.Log($"[Before Deletion] {name} has {transform.childCount} children.");

        // Find and remove existing child prefab
        for (int i = transform.childCount - 1; i >= 0; i--)
        {
            Debug.Log($"Destroying child: {transform.GetChild(i).gameObject.name}");
            DestroyImmediate(transform.GetChild(i).gameObject);
        }

        Debug.Log($"[After Deletion] {name} now has {transform.childCount} children.");

        // Instantiate new prefab as a child
        GameObject selectedPrefab = config.prefabs[selectedPrefabIndex];
        if (selectedPrefab)
        {
            GameObject newInstance = (GameObject)PrefabUtility.InstantiatePrefab(selectedPrefab);
            newInstance.transform.SetParent(transform, false); // Keep world transform settings

            Debug.Log($"New prefab instantiated: {newInstance.name}");

            ApplyMaterialProperties();
        }
    }

    private void ApplyMaterialProperties()
    {
        Renderer rend = GetComponentInChildren<Renderer>(); // Get the renderer from the child prefab
        if (rend != null)
        {
            MaterialPropertyBlock mpb = new MaterialPropertyBlock();
            rend.GetPropertyBlock(mpb);

            if (config.textures.Count > selectedTextureIndex)
                mpb.SetTexture("_MainTex", config.textures[selectedTextureIndex]);

            if (config.colors.Count > selectedColorIndex)
                mpb.SetColor("_Color", config.colors[selectedColorIndex]);

            rend.SetPropertyBlock(mpb);
        }
    }
}


```
