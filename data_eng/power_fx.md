// --- 7.4 Final resync UI with latest active refs ---

If(
    varHasModuleContext,
    With(
        {
            // pull fresh active refs for this module from SharePoint after all patches
            colFreshActiveRefs:
                Filter(
                    Skill_Matrix_Reference,
                    Mod_ID = varModuleIdSafe && IsActive = true
                )
        },
        // Rebuild colModuleCatItems_Working to reflect only what's ACTIVE right now
        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                colFreshActiveRefs,
                // expose CatItem_ID for checkboxes & gallery
                CatItem_ID,
                // expose Skill_Type displayed in the grid
                Skill_Type,
                // expose Category/Item labels so the gallery can still render
                Category,
                Item
            )
        )
    );
);



