ClearCollect(
    colModuleCatItems_Working,
    AddColumns(
        Filter(
            Skill_Matrix_Reference,
            Mod_ID = varModuleIdSafe && IsActive = true
        ),
        "Category", Category,
        "Item", Item,
        "Skill_Type", Skill_Type
    )
);
