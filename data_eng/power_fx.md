// 2.2 Remove any links that are no longer checked

// Step 1: collect the refs to delete into a local collection
ClearCollect(
    colRefsToDelete,
    Filter(
        Skill_Matrix_Reference,
        Mod_ID = varSelectedModuleId &&
        CountIf(colModuleCatItems_Working, CatItem_ID = CatItem_ID) = 0
    )
);

// Step 2: remove them in one go (now it's outside of the ForAll)
If(
    CountRows(colRefsToDelete) > 0,
    RemoveIf(
        Skill_Matrix_Reference,
        Reference_ID in colRefsToDelete.Reference_ID
    )
);
