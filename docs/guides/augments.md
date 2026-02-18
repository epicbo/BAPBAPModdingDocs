# Augments

## Overview

BAPBAP uses a class called **BAPBAP.Entities.Passive** to apply a passive modifier to an entity. Every augment is a passive and inherits from this class, but not every passive is an augment. We also have a class called **BAPBAP.Entities.PassiveSO** (a passive scriptable object) which hold the configuration for every passive. Whenever a passive is applied to an entity (for example when a player select an augment), the **PassiveSO** object is used to create the **Passive** instance that is then applied to the entity. In other words, there are only ever one instance of **PassiveSO** for each type of passive, but there can be multiple instances of **Passive** at once. Of course, **Passive** and **PassiveSO** are just base classes, the actual effect of the passives are implemented in child classes of these two. 

Every passive has a unique id (_passiveId_). Often we only have the _passiveId_ to work with and need to retrieve the underlying **PassiveSO** object. Luckily there is the **BAPBAP.Local.PassiveManager** class which is often quite accessible, that in turn has a **BAPBAP.Entities.PassiveLibrary** object (_library_) that finally has an array called _passives_ that acts as a master list for every passive in the game. For example, if we want to add a new augment to the game, we would have add them to this array (more on this later).

Another class that is good to know about is **BAPBAP.Entities.CharPassives**. It is attached to a specific character and handles things such as adding, removing and triggering effects of passives on that character. Very useful if we want to make our own augments. Finally, there's also **BAPBAP.Local.AugmentManager** which handles information that is specific to augments, such as, which augments are character specific, which require previous augments (eg. you can't get Firewave Cooldown before having Firewave) and so on. Also very important if we want to add more augments.

## Hidden augments

The game comes with a bunch of augments that exist in the code but are not enabled in game. Luckily it's fairly easy to enable them. Since they're already in the game, only the host needs the mod, which is a nice bonus. Worth noting is that the polish of them vary greatly. Some are are fully functional and can be added without issue, some work but may not have the best icon/description/visuals/balance and some are completely broken. Try them out and see which ones you like!

Step one is figuring out the _passiveId_ of the augments we are interested in using. In this example, we will use ID 247, called "Critical Ability" which lets us critically hit with abilities and adds a small amount of crit chance. 

Next step is patching **AugmentManager.PreAwake**. I found this method useful for this purpose because it is only called once, and at that point all augments have already been initialized, but we can still make changes to them. It is important to understand how augments are selected ingame. There are multiple "augment pools" that the game selects from. _Starter_, _Character_, _Wildcard_ and _Fallback_. When you get an option to select an augment, they are picked as such:

- One augment is picked from _Starter_
- One augment is picked from _Wildcard_
- One augment is either from _Starter_ or _Character_ (50% for each)

If there is no possible augment to pick for a specific augment slot, then one is picked from the _Fallback_ pool (you may have seen some of the really weak ones that give 7% attack speed and such). This can happen if there are no augments available left in that pool to pick. With this information in mind, we have the following fields in **AugmentManger**:

- _allAugmentIds_: A list of _passiveId_ of all available augments. Actually, I am not sure where this is being used or whether we actually need to modify it, doesn't hurt to.
- _charAugmentIds_: A 2D array where the first index refers to the _charId_ of the character, the second to the _passiveId_. This one is used when creating a game and setting which augments should be available. Adding an augment to this array means that it will show up in that menu under the that specific character. It is also used to build the list of character specific augments in the Training mode.
- _genericAugmentIds_: Similar to _charAugmentIds_, but for the non-character specific augments.
- _selectionFallbackAugments_: Here we have a list of augments in the _Fallback_ pool. Note that these are **PassiveSO** and not IDs.
- _startingAugmentTrees_: The augments in the _Starter_ pool, as **AugmentManager.AugmentTree** objects. The _augment_ field is the augment itself (again as **PassiveSO** and not and ID). There are also some additional fields that can be useful, such as _requiredCharacter_ if you only want specific characters to be able to obtain it, _incompatibleAugments_ and also _childrenAugments_, which will become available when the player obtains the augment (think Firewave Cooldown after obtaining Firewave).
- _wildCardAugmentTree_: Very similar to _startingAugmentTree_, but for the _Wildcard_ pool instead.
- _wildCardRareAugmentTree_: Not 100% sure what the deal is with this one, but I guess it was used in another game mode. The game seems to pick a Wildcard augment from either of them (if it can't find a valid augment in one of the wildcard trees, it will just select the other). Might as well add wildcard augments to this one just to be sure.

Here's what the code could look like (I went with adding the augment to the _Starter_ pool).

```csharp
[HarmonyPatch(typeof(PassiveManager), nameof(PassiveManager.PreAwake))]
public class HiddenAugmentPatch
{
    public static void Postfix(AugmentManager _instance)
    {
        var passiveId = 247;

        // Get a reference to the PassiveManager. All managers are available from LocalManager.Instance!
        var passiveManager = LocalManager.Instance.passiveManager;

        // Fetch the PassiveSO from the PassiveManager
        var newAugment = passiveManager.library.passives[passiveId];

        // We need a new AugmentTree object, but we'll be lazy and just clone an existing one
        var newAugmentTree = _instance.startingAugmentTrees[0].MemberwiseClone().Cast<AugmentManager.AugmentTree>();

        // Let's clear its children by replacing the value with an empty array. 
        newAugmentTree.childrenAugments = new Il2CppReferenceArray<AugmentManager.DependencyAugment>(0);

        // Set the augment in the augment tree to our new augment
        newAugmentTree.augment = newAugment;

        // Add the passiveId of our agument to allAugmentIds and genericAugmentIds. 
        // Note: since these fields are arrays (of type Il2CppStructArray<int>), you'll have to create a new one of 'length + 1',
        // copy the values from the old one, add the new passiveId, and then override the old one. 
        // I am using a helper I wrote for this.
        _instance.allAugmentIds = Util.AppendIntArray(_instance.allAugmentIds, passiveId);
        _instance.genericAugmentIds = Util.AppendIntArray(_instance.genericAugmentIds, passiveId);

        // Similar to above, but here we're adding our new augment tree to the starting augment tree pool.
        _instance.startingAugmentTrees = Util.AppendReferenceArray(_instance.startingAugmentTrees, newAugmentTree);
    }
}
```




## Modifying existing augments

TODO

## Custom augments

TODO