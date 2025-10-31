ClearCollect(
    colModuleCatItems_Working,
    Filter(
        Skill_Matrix_Reference,
        Mod_ID = varModuleIdSafe && IsActive = true
    )
);
